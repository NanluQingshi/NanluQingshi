# Claude Code 源码深度分析与学习指南

> 分析时间：2026-05-11
> 项目路径：`/Users/nlqs/Documents/code/original_study/claude-code`

---

## 一、项目是什么

**Claude Code** 是 Anthropic 官方发布的命令行 AI 编程助手（CLI）。你在终端里运行 `claude`，它就是这个程序。

**解决的核心问题：** 让 AI 真正能操控你的开发环境——读写文件、执行命令、搜索代码、调用 API、管理 Git——而不仅仅是聊天。

**规模：** ~1,900 个文件、512,000+ 行 TypeScript 代码。

---

## 二、技术栈总览

| 层次 | 技术 | 说明 |
|------|------|------|
| 运行时 | **Bun** | 替代 Node.js，性能更高，也是构建工具 |
| 语言 | **TypeScript（严格模式）** | 全量类型覆盖 |
| 终端 UI | **React + Ink** | 用 React 写终端界面，Ink 是适配层 |
| CLI 解析 | **Commander.js** | 解析命令行参数和子命令 |
| 数据验证 | **Zod v4** | 工具输入参数的运行时验证 |
| AI API | **Anthropic SDK** | 与 Claude 模型通信 |
| 状态管理 | **自研 Zustand 风格 Store** | 简单全局状态，无 Redux |
| 外部工具协议 | **MCP（Model Context Protocol）** | 接入第三方工具服务器 |
| 代码搜索 | **ripgrep** | 比 grep 快几十倍的文件搜索 |
| 特性开关 | **Bun feature flags + GrowthBook** | 编译时死代码消除 + 运行时 A/B |
| 认证 | **OAuth 2.0 + macOS Keychain** | 安全存储 API Key |

---

## 三、目录结构（按职责分组）

```
src/
│
├── 【核心入口】
│   ├── main.tsx                 # CLI 入口，Commander.js 初始化
│   ├── entrypoints/cli.tsx      # CLI 模式启动
│   ├── entrypoints/init.ts      # 所有初始化逻辑
│   └── replLauncher.tsx         # 启动交互式 REPL
│
├── 【查询引擎（最核心）】
│   ├── QueryEngine.ts           # 46K 行，LLM 对话+工具循环的核心
│   ├── query.ts                 # 查询管道，API 调用
│   └── context.ts               # 系统上下文收集（cwd、git 状态等）
│
├── 【工具系统】
│   ├── Tool.ts                  # 29K 行，所有工具的接口和类型定义
│   ├── tools.ts                 # 工具注册表，条件加载
│   └── tools/                   # 45 个工具的具体实现
│       ├── BashTool/            # Shell 命令执行
│       ├── FileReadTool/        # 读文件（支持图片/PDF/Notebook）
│       ├── FileWriteTool/       # 写文件
│       ├── FileEditTool/        # 局部编辑（字符串替换）
│       ├── GlobTool/            # 文件路径模式匹配
│       ├── GrepTool/            # 基于 ripgrep 的内容搜索
│       ├── WebFetchTool/        # 获取 URL 内容
│       ├── WebSearchTool/       # 网页搜索
│       ├── AgentTool/           # 生成并管理子代理
│       ├── SkillTool/           # 执行技能工作流
│       ├── MCPTool/             # 调用 MCP 工具服务器
│       ├── TaskCreateTool/      # 创建任务
│       ├── CronCreateTool/      # 定时触发（特性标志控制）
│       └── EnterWorktreeTool/   # Git worktree 隔离
│
├── 【命令系统】
│   ├── commands.ts              # 25K 行，斜杠命令注册表
│   └── commands/                # ~103 个 /slash 命令实现
│       ├── commit.js
│       ├── review.js
│       ├── config/
│       ├── mcp/
│       └── ...
│
├── 【终端 UI（React + Ink）】
│   ├── components/              # ~150 个 Ink 组件
│   │   ├── App.tsx              # 顶层容器
│   │   ├── REPL.tsx（screens/） # 主 REPL 界面
│   │   ├── Message.tsx          # 单条消息渲染
│   │   ├── PromptInput.tsx      # 用户输入框
│   │   └── Spinner.tsx          # 加载动画
│   └── hooks/                   # ~87 个自定义 React Hooks
│       ├── useCanUseTool.tsx
│       └── toolPermission/
│
├── 【全局状态】
│   ├── state/AppStateStore.ts   # AppState 类型定义（真相之源）
│   ├── state/store.ts           # store 工厂（Zustand 风格）
│   └── state/AppState.tsx       # React Context Provider
│
├── 【外部服务】
│   └── services/
│       ├── api/claude.js        # Anthropic API 客户端封装
│       ├── mcp/                 # MCP 协议客户端
│       ├── oauth/               # OAuth 2.0 认证流
│       ├── lsp/                 # Language Server Protocol
│       ├── compact/             # 上下文压缩引擎
│       ├── analytics/           # GrowthBook 特性开关
│       └── tokenEstimation.ts   # Token 数量估算
│
├── 【IDE 桥接】
│   └── bridge/                  # 33 个文件，与 VS Code/JetBrains 通信
│
├── 【插件与技能】
│   ├── plugins/bundled/         # 内置插件
│   ├── skills/bundled/          # 内置技能（可重用工作流）
│   └── memdir/                  # CLAUDE.md 内存文件处理
│
├── 【配置与权限】
│   ├── utils/permissions/       # 权限规则评估
│   ├── utils/settings/          # 配置加载、迁移
│   └── schemas/                 # Zod 配置模式定义
│
└── 【工具函数（最大目录）】
    └── utils/                   # 331 个文件，各类辅助函数
```

---

## 四、核心架构：执行流程

### 4.1 启动流程

```
用户运行 `claude` 命令
        │
        ▼
main.tsx                         ← 入口
  ├─ 并行启动优化（IO 预热）
  │  ├─ startMdmRawRead()        # 读 MDM 组织策略（macOS）
  │  └─ startKeychainPrefetch()  # 读 API Key 缓存
  │
  ├─ Commander.js 解析参数
  │
  └─ init()                      ← 初始化阶段
     ├─ 加载全局配置
     ├─ 运行迁移脚本（models 版本升级）
     ├─ 初始化 GrowthBook（特性开关）
     ├─ 初始化 MCP 客户端
     └─ 初始化 LSP 服务器
        │
        ▼
     launchRepl()                ← 启动交互界面
     ├─ 创建 AppState store
     ├─ 创建 QueryEngine 实例
     └─ React+Ink 渲染 REPL 屏幕
```

### 4.2 核心查询循环（最重要）

这是整个项目最核心的部分，理解它就理解了 Claude Code 的工作原理：

```
用户输入消息
        │
        ▼
QueryEngine.submitMessage()
  ├─ 构建系统提示（tools 描述、权限模式、CLAUDE.md 内存）
  │
  ▼
┌─────────────────────────────────────────────────────┐
│                    工具调用循环                       │
│                                                      │
│  1. normalizeMessagesForAPI()  格式化消息历史        │
│  2. fetchMessage()             调用 Claude API      │
│  3. 检查 stop_reason                                 │
│     ├─ "end_turn"     → 完成，返回给用户             │
│     └─ "tool_use"     → 需要执行工具                │
│         │                                            │
│         ▼                                            │
│  4. 权限检查 canUseTool()                            │
│     ├─ 检查规则白名单/黑名单                        │
│     ├─ 询问用户（如果需要）                         │
│     └─ 返回 allow/deny                              │
│         │                                            │
│  5. 执行工具 tool.call(args)                        │
│     └─ 返回 ToolResult                              │
│         │                                            │
│  6. 将 ToolResult 加入消息历史                       │
│  7. 返回步骤 1，继续循环                             │
└─────────────────────────────────────────────────────┘
```

---

## 五、关键设计模式

### 5.1 并行 I/O 预取（启动优化）

```typescript
// main.tsx 中，在加载重型模块前先把 I/O 发出去
startMdmRawRead()           // 异步读 macOS MDM 配置
startKeychainPrefetch()     // 异步预热 Keychain 访问
// 然后正常加载其他模块
// 等到真正需要这些值时，它们已经就绪了
```

**学习价值：** 启动性能优化的标准做法——把必须等的 I/O 提前发出，而不是串行等待。

### 5.2 编译时死代码消除

```typescript
import { feature } from 'bun:bundle'

if (feature('VOICE_MODE')) {
  // Bun 编译时如果 VOICE_MODE=false，这段代码完全不会打包进去
  registerVoiceTool()
}
```

**学习价值：** 用特性开关控制功能，同时不牺牲包体积。

### 5.3 工具注册模式

```typescript
// tools.ts：工具是数组，按条件动态组合
function getTools(context): Tool[] {
  const tools = [BashTool, FileReadTool, FileWriteTool, ...]  // 始终包含
  
  if (feature('AGENT_TRIGGERS')) tools.push(CronCreateTool)
  if (context.isMcpEnabled) tools.push(...mcpTools)
  
  return tools
}
```

**学习价值：** 无需改动核心代码，工具即插即用。

### 5.4 权限系统分层

```
PermissionMode:
  "default"     → 每次询问用户
  "auto"        → 用 AI 分类器判断是否安全，自动执行
  "bypassPerms" → 完全跳过（谨慎使用）

规则系统:
  alwaysAllowRules   → ["Bash(git status)", "Read(**/*.ts)"]
  alwaysDenyRules    → ["Bash(rm -rf *)"]
  alwaysAskRules     → ["Write(*.json)"]
```

**学习价值：** 安全与效率平衡的多级权限设计。

### 5.5 React + Ink 终端 UI

```tsx
// 终端里的 "组件" 就是 React 组件
function REPL() {
  return (
    <Box flexDirection="column">
      <Messages messages={messages} />
      <Spinner isLoading={isProcessing} />
      <PromptInput onSubmit={handleSubmit} />
    </Box>
  )
}
```

**学习价值：** 用熟悉的 React 思维写终端 UI，而不是手动控制光标位置。

---

## 六、建议的学习路径

### 第 1 阶段：搭建认知框架（1-2 天）

先不看代码，搞清楚 **"这个系统做什么、怎么工作"**。

1. 读本文档，理解整体架构
2. 安装并实际使用 Claude Code，体验用户视角
3. 在终端里用 `claude --help` 看命令结构
4. 理解 [工具循环] 是核心——AI 调用工具，工具返回结果，AI 继续

### 第 2 阶段：入口到核心（2-3 天）

按执行顺序追踪主流程：

```
① src/main.tsx                  # CLI 如何启动
       ↓
② src/entrypoints/init.ts       # 初始化了什么
       ↓
③ src/replLauncher.tsx          # REPL 如何启动
       ↓
④ src/QueryEngine.ts            # 查询引擎核心（重点！）
       ↓
⑤ src/query.ts                  # API 调用细节
```

**每读一个文件，问自己：**
- 这个文件的职责是什么？
- 它依赖谁、被谁依赖？
- 最重要的函数/类是什么？

### 第 3 阶段：读一个完整工具（2-3 天）

选最简单的工具开始，完整读一个工具的实现：

**推荐顺序：**

```
① src/tools/GrepTool/         # 最简单，只是调用 ripgrep
② src/tools/FileReadTool/     # 稍复杂，支持多格式
③ src/tools/BashTool/         # 有并发安全、沙箱等概念
④ src/tools/AgentTool/        # 最复杂，多代理协调
```

**读工具时关注：**
- `inputSchema`（Zod 定义）：工具接受什么参数
- `call()` 方法：工具怎么执行
- 返回的 `ToolResult`：结果结构是什么

### 第 4 阶段：读权限系统（2 天）

权限系统是 Claude Code 安全性的核心：

```
src/hooks/useCanUseTool.tsx
src/hooks/toolPermission/
src/utils/permissions/
```

理解：
- 权限规则如何定义和匹配
- 用户怎么授权工具执行
- Auto 模式的 AI 分类器如何工作

### 第 5 阶段：读 UI 层（2 天）

```
src/screens/REPL.tsx          # 主界面
src/components/Message.tsx    # 消息渲染
src/components/PromptInput.tsx # 输入框
src/state/                    # 全局状态
```

重点理解：
- AppState 的数据结构
- store 的更新机制
- React 状态如何与查询引擎联动

### 第 6 阶段：深入子系统（按兴趣选择）

根据你的兴趣选择性深入：

| 子系统 | 路径 | 难度 |
|--------|------|------|
| MCP 协议集成 | `src/services/mcp/` | ★★★ |
| IDE 桥接通信 | `src/bridge/` | ★★★★ |
| 上下文压缩 | `src/services/compact/` | ★★★ |
| 多代理协调 | `src/tools/AgentTool/` + `src/coordinator/` | ★★★★★ |
| 技能系统 | `src/skills/` | ★★ |
| OAuth 认证 | `src/services/oauth/` | ★★★ |

---

## 七、学习时的关键问题清单

读每个模块时，尝试回答这些问题：

**关于设计：**
- 为什么这样设计，而不是更简单的方式？
- 这里解决了什么权衡（性能 vs 安全？灵活 vs 简单？）

**关于工具系统：**
- 新增一个工具需要改哪些文件？
- 工具如何获得权限提示的？

**关于查询引擎：**
- 一次对话的消息历史是如何在内存中表示的？
- 工具结果超过 context window 怎么办（压缩机制）？
- 多个工具同时调用时怎么协调？

**关于 UI：**
- 流式响应（streaming）是如何在终端实时显示的？
- 用户中断（Ctrl+C）时发生了什么？

---

## 八、值得重点学习的代码片段

### 1. 系统提示构建（理解 AI 工作方式）
在 `QueryEngine.ts` 中搜索 `buildSystemPrompt`——这里能看到怎么把工具描述、权限说明、用户上下文组装成给 AI 看的系统提示。

### 2. 流式响应处理
在 `services/api/claude.js` 中搜索 `stream` 相关代码——看 Claude API 的流式事件如何被处理成增量更新。

### 3. Zod 工具定义
任意一个工具（如 `src/tools/GrepTool/`）——看 `inputSchema` 如何用 Zod 定义工具参数，这会被自动转为 JSON Schema 发给 AI。

### 4. 权限规则评估
`src/utils/permissions/` 中的规则匹配逻辑——理解 glob 模式如何匹配工具调用。

### 5. 消息规范化
`query.ts` 中的 `normalizeMessagesForAPI()`——看消息历史在发给 API 前经历了哪些处理。

---

## 九、调试技巧

学习源码时，结合运行帮助理解：

```bash
# 1. 打印详细日志
CLAUDE_DEBUG=1 claude

# 2. 使用 headless 模式观察行为
claude --print "list files in current directory"

# 3. 查看工具执行
# 在 BashTool/call() 里加 console.log，观察参数

# 4. 追踪 API 调用
# 在 services/api/claude.js 里加日志，看实际发送了什么
```

---

## 十、核心类型速查

理解这些类型，就掌握了整个系统的"语言"：

```typescript
// 一条消息
type Message = UserMessage | AssistantMessage | ToolResultMessage

// 一个工具
interface Tool<TInput, TOutput> {
  name: string
  description: string
  inputSchema: ZodSchema<TInput>        // 参数定义（自动转 JSON Schema）
  call(input: TInput): Promise<ToolResult<TOutput>>
}

// 工具执行结果
type ToolResult<T> = {
  data: T
  newMessages?: Message[]             // 工具可以注入额外消息
  contextModifier?: (ctx) => ctx      // 工具可以修改执行上下文
}

// 全局应用状态（真相之源）
type AppState = {
  messages: Message[]
  model: string
  isProcessing: boolean
  mcpClients: McpClient[]
  permissionMode: PermissionMode
  // ... 更多字段
}

// 权限模式
type PermissionMode = 'default' | 'auto' | 'plan' | 'bypassPerms'
```

---

## 十一、参考资源

- **官方文档：** `https://docs.anthropic.com/claude-code`
- **MCP 协议：** `https://modelcontextprotocol.io`
- **Ink（React for Terminal）：** `https://github.com/vadimdemedes/ink`
- **Bun 文档：** `https://bun.sh/docs`

---

*这份文档是学习起点，不是终点。读代码时带着问题去读，比逐行扫描有效 10 倍。*
