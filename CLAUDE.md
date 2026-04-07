# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 提供在操作本代码仓库时的指导。

## 项目概述

这是 Anthropic 官方 Claude Code CLI 工具的**逆向工程/反编译**版本。目标是在精简次要功能的同时恢复核心功能。许多模块被存根化或通过功能标志关闭。代码库有约 1341 个来自反编译的 tsc 错误（主要是 `unknown`/`never`/`{}` 类型）——这些**不会**阻止 Bun 运行时执行。

## 命令

```bash
# 安装依赖
bun install

# 开发模式（通过 -d 标志注入 MACRO defines 运行 cli.tsx）
bun run dev

# 带调试器的开发模式（设置 BUN_INSPECT=9229 指定端口）
bun run dev:inspect

# 管道模式
echo "say hello" | bun run src/entrypoints/cli.tsx -p

# 构建（代码分割，输出 dist/cli.js + ~450 个 chunk 文件）
bun run build

# 测试
bun test                  # 运行所有测试
bun test src/utils/__tests__/hash.test.ts   # 运行单个文件
bun test --coverage       # 带覆盖率报告

# 代码检查与格式化（Biome）
bun run lint              # 仅检查
bun run lint:fix          # 自动修复
bun run format            # 格式化所有 src/ 文件

# 健康检查
bun run health

# 检查未使用的导出
bun run check:unused

# 文档开发服务器（Mintlify）
bun run docs:dev
```

### 网络问题（中国）

如果 GitHub 下载失败，安装前设置此项：
```bash
export DEFAULT_RELEASE_BASE=https://ghproxy.net/https://github.com/microsoft/ripgrep-prebuilt/releases/download/v15.0.1
```

详细的测试规范、覆盖状态和改进计划见 `docs/testing-spec.md`。

## 架构

### 运行时与构建

- **运行时**: Bun（非 Node.js）。所有导入、构建和执行都使用 Bun API。
- **构建**: `build.ts` 执行 `Bun.build()` 并启用 `splitting: true`，入口为 `src/entrypoints/cli.tsx`，输出 `dist/cli.js` + chunk 文件。默认启用 `AGENT_TRIGGERS_REMOTE`、`CHICAGO_MCP`、`VOICE_MODE` 功能。构建后自动替换 `import.meta.require` 为 Node.js 兼容版本（产物可通过 bun 或 node 运行）。
- **开发模式**: `scripts/dev.ts` 通过 Bun `-d` 标志注入 `MACRO.*` defines，运行 `src/entrypoints/cli.tsx`。默认启用 `BUDDY`、`TRANSCRIPT_CLASSIFIER`、`BRIDGE_MODE`、`AGENT_TRIGGERS_REMOTE`、`CHICAGO_MCP`、`VOICE_MODE` 六个功能。看到版本号 888 说明环境正确。
- **模块系统**: ESM（`"type": "module"`），使用 `react-jsx` 转换的 TSX。
- **Monorepo**: Bun workspaces — 内部包位于 `packages/`，通过 `workspace:*` 解析。
- **代码检查/格式化**: Biome（`biome.json`）。`bun run lint` / `bun run lint:fix` / `bun run format`。
- **Defines**: 集中管理在 `scripts/defines.ts`。当前版本 `2.1.888`，版本历史见 `docs/` 目录。

### 入口与启动

1. **`src/entrypoints/cli.tsx`** — 真正的入口点。`main()` 函数按优先级处理多条快速路径：
   - `--version` / `-v` — 零模块加载
   - `--dump-system-prompt` — 功能门控 (DUMP_SYSTEM_PROMPT)
   - `--claude-in-chrome-mcp` / `--chrome-native-host`
   - `--daemon-worker=<kind>` — 功能门控 (DAEMON)
   - `remote-control` / `rc` / `bridge` — 功能门控 (BRIDGE_MODE)
   - `daemon` — 功能门控 (DAEMON)
   - `ps` / `logs` / `attach` / `kill` / `--bg` — 功能门控 (BG_SESSIONS)
   - `--tmux` + `--worktree` 组合
   - 默认路径：加载 `main.tsx` 启动完整 CLI
2. **`src/main.tsx`** (~4680 行) — Commander.js CLI 定义。注册大量子命令：`mcp` (serve/add/remove/list...)、`server`、`ssh`、`open`、`auth`、`plugin`、`agents`、`auto-mode`、`doctor`、`update` 等。主 `.action()` 处理器负责权限、MCP、会话恢复、REPL/Headless 模式分发。
3. **`src/entrypoints/init.ts`** — 一次性初始化（遥测、配置、信任对话框）。

### 核心循环

- **`src/query.ts`** — 主要的 API 查询函数。向 Claude API 发送消息，处理流式响应，处理工具调用，管理对话轮次循环。
- **`src/QueryEngine.ts`** — 包装 `query()` 的更高层编排器。管理对话状态、压缩、文件历史快照、归属和轮次级别的簿记。REPL 界面使用。
- **`src/screens/REPL.tsx`** — 交互式 REPL 界面（React/Ink 组件）。处理用户输入、消息显示、工具权限提示和键盘快捷键。

### API 层

- **`src/services/api/claude.ts`** — 核心 API 客户端。构建请求参数（系统提示、消息、工具、beta），调用 Anthropic SDK 流式端点，处理 `BetaRawMessageStreamEvent` 事件。
- 支持多个提供商：Anthropic 直连、AWS Bedrock、Google Vertex、Azure。
- 提供商选择位于 `src/utils/model/providers.ts`。

### 工具系统

- **`src/Tool.ts`** — 工具接口定义（`Tool` 类型）和工具函数（`findToolByName`、`toolMatchesName`）。
- **`src/tools.ts`** — 工具注册表。组装工具列表；某些工具通过 `feature()` 标志或 `process.env.USER_TYPE` 条件加载。
- **`src/tools/<ToolName>/`** — 61 个工具目录（如 BashTool、FileEditTool、GrepTool、AgentTool、WebFetchTool、LSPTool、MCPTool 等）。每个工具包含 `name`、`description`、`inputSchema`、`call()` 及可选的 React 渲染组件。
- **`src/tools/shared/`** — 工具共享工具函数。

### UI 层（Ink）

- **`src/ink.ts`** — 带 ThemeProvider 注入的 Ink 渲染包装器。
- **`src/ink/`** — 自定义 Ink 框架（fork/内部）：自定义协调器、hooks（`useInput`、`useTerminalSize`、`useSearchHighlight`）、虚拟列表渲染。
- **`src/components/`** — 大量 React 组件（170+ 项），渲染于终端 Ink 环境中。关键组件：
  - `App.tsx` — 根提供者（AppState、Stats、FpsMetrics）
  - `Messages.tsx` / `MessageRow.tsx` — 对话消息渲染
  - `PromptInput/` — 用户输入处理
  - `permissions/` — 工具权限批准 UI
  - `design-system/` — 复用 UI 组件（Dialog、FuzzyPicker、ProgressBar、ThemeProvider 等）
- 组件使用 React Compiler 运行时（`react/compiler-runtime`）— 反编译输出中到处都有 `_c()` 记忆化调用。

### 状态管理

- **`src/state/AppState.tsx`** — 中央应用状态类型和上下文提供者。包含消息、工具、权限、MCP 连接等。
- **`src/state/AppStateStore.ts`** — 默认状态和存储工厂。
- **`src/state/store.ts`** — AppState 的 Zustand 风格存储（`createStore`）。
- **`src/state/selectors.ts`** — 状态选择器。
- **`src/bootstrap/state.ts`** — 会话全局状态的模块级单例（会话 ID、CWD、项目根目录、token 计数、模型覆盖、客户端类型、权限模式）。

### Bridge / 远程控制

- **`src/bridge/`** (~35 个文件) — 远程控制 / Bridge 模式。由 `BRIDGE_MODE` 功能门控。包含 bridge API、会话管理、JWT 认证、消息传输、权限回调等。入口：`bridgeMain.ts`。
- CLI 快速路径：`claude remote-control` / `claude rc` / `claude bridge`。

### 守护进程模式

- **`src/daemon/`** — 守护进程模式（常驻 supervisor）。由 `DAEMON` 功能门控。包含 `main.ts`（入口）和 `workerRegistry.ts`（worker 管理）。

### 上下文与系统提示

- **`src/context.ts`** — 为 API 调用构建系统/用户上下文（git 状态、日期、CLAUDE.md 内容、记忆文件）。
- **`src/utils/claudemd.ts`** — 从项目层次结构发现和加载 CLAUDE.md 文件。

### 功能标志系统

功能标志控制运行时启用的功能：

- **在代码中使用**: 统一通过 `import { feature } from 'bun:bundle'` 导入，调用 `feature('FLAG_NAME')` 返回 `boolean`。**不要**在 `cli.tsx` 或其他文件里自己定义 `feature` 函数或覆盖这个 import。
- **启用方式**: 通过环境变量 `FEATURE_<FLAG_NAME>=1`。例如 `FEATURE_BUDDY=1 bun run dev` 启用 BUDDY 功能。
- **开发默认功能**: `BUDDY`、`TRANSCRIPT_CLASSIFIER`、`BRIDGE_MODE`、`AGENT_TRIGGERS_REMOTE`、`CHICAGO_MCP`、`VOICE_MODE`（见 `scripts/dev.ts`）。
- **构建默认功能**: `AGENT_TRIGGERS_REMOTE`、`CHICAGO_MCP`、`VOICE_MODE`（见 `build.ts`）。
- **常见标志**: `BUDDY`、`DAEMON`、`BRIDGE_MODE`、`BG_SESSIONS`、`PROACTIVE`、`KAIROS`、`VOICE_MODE`、`FORK_SUBAGENT`、`SSH_REMOTE`、`DIRECT_CONNECT`、`TEMPLATES`、`CHICAGO_MCP`、`BYOC_ENVIRONMENT_RUNNER`、`SELF_HOSTED_RUNNER`、`COORDINATOR_MODE`、`UDS_INBOX`、`LODESTONE`、`ABLATION_BASELINE` 等。
- **类型声明**: `src/types/internal-modules.d.ts` 中声明了 `bun:bundle` 模块的 `feature` 函数签名。

**新增功能的正确做法**: 保留 `import { feature } from 'bun:bundle'` + `feature('FLAG_NAME')` 的标准模式，在运行时通过环境变量或配置控制，不要绕过功能标志直接 import。

### 存根/已删除模块

| 模块 | 状态 |
|------|------|
| Computer Use (`@ant/*`) | 已恢复 — `computer-use-swift`、`computer-use-input`、`computer-use-mcp`、`claude-for-chrome-mcp` 均有完整实现，macOS + Windows 可用，Linux 后端待完成 |
| `*-napi` packages | `audio-capture-napi`、`image-processor-napi` 已恢复实现；`color-diff-napi` 完整实现；`url-handler-napi`、`modifiers-napi` 仍为存根 |
| Voice Mode | 已恢复 — `src/voice/`、`src/hooks/useVoiceIntegration.tsx`、`src/services/voiceStreamSTT.ts` 等，Push-to-Talk 语音输入（需 Anthropic OAuth） |
| OpenAI 兼容层 | 已恢复 — `src/services/api/openai/`，支持 Ollama/DeepSeek/vLLM 等任意 OpenAI 协议端点，通过 `CLAUDE_CODE_USE_OPENAI=1` 启用 |
| Analytics / GrowthBook / Sentry | 空实现 |
| Magic Docs / LSP Server | 已移除 |
| Plugins / Marketplace | 已移除 |
| MCP OAuth | 已简化 |

### Computer Use

功能标志 `CHICAGO_MCP`，开发/构建默认启用。实现跨平台屏幕操控（macOS + Windows 可用，Linux 待完成）。

- **`packages/@ant/computer-use-mcp/`** — MCP server，注册截图/键鼠/剪贴板/应用管理工具
- **`packages/@ant/computer-use-input/`** — 键鼠模拟，dispatcher + 各平台后端（`backends/darwin.ts`、`win32.ts`、`linux.ts`）
- **`packages/@ant/computer-use-swift/`** — 截图 + 应用管理，同样 dispatcher + 各平台后端
- **`packages/@ant/claude-for-chrome-mcp/`** — Chrome 浏览器控制（独立于 Computer Use，通过 `--chrome` CLI 参数启用）

详见 `docs/features/computer-use.md`。

### Voice Mode

功能标志 `VOICE_MODE`，开发/构建默认启用。Push-to-Talk 语音输入，音频通过 WebSocket 流式传输到 Anthropic STT（Nova 3）。需要 Anthropic OAuth（非 API key）。

- **`src/voice/voiceModeEnabled.ts`** — 三层门控（功能标志 + GrowthBook + OAuth auth）
- **`src/hooks/useVoice.ts`** — React hook 管理录音状态和 WebSocket 连接
- **`src/services/voiceStreamSTT.ts`** — STT WebSocket 流式传输

详见 `docs/features/voice-mode.md`。

### OpenAI 兼容层

通过 `CLAUDE_CODE_USE_OPENAI=1` 环境变量启用，支持任意 OpenAI Chat Completions 协议端点（Ollama、DeepSeek、vLLM 等）。流适配器模式：在 `queryModel()` 中将 Anthropic 格式请求转为 OpenAI 格式，再将 SSE 流转换回 `BetaRawMessageStreamEvent`，下游代码完全不改。

- **`src/services/api/openai/`** — 客户端、消息/工具转换、流适配、模型映射
- **`src/utils/model/providers.ts`** — 添加 `'openai'` 提供商类型（最高优先级）

关键环境变量：`CLAUDE_CODE_USE_OPENAI`、`OPENAI_API_KEY`、`OPENAI_BASE_URL`、`OPENAI_MODEL`、`OPENAI_DEFAULT_OPUS_MODEL`、`OPENAI_DEFAULT_SONNET_MODEL`、`OPENAI_DEFAULT_HAIKU_MODEL`。详见 `docs/plans/openai-compatibility.md`。

### Gemini 兼容层

通过 `CLAUDE_CODE_USE_GEMINI=1` 环境变量或 `modelType: "gemini"` 设置启用，支持 Google Gemini API。独立的环境变量体系，不与 OpenAI 或 Anthropic 配置混杂。

- **`src/services/api/gemini/`** — 客户端、模型映射、类型定义
- **`src/utils/model/providers.ts`** — 添加 `'gemini'` 提供商类型
- **`src/utils/managedEnvConstants.ts`** — Gemini 专用的托管环境变量

关键环境变量：
- `CLAUDE_CODE_USE_GEMINI` - 启用 Gemini 提供商
- `GEMINI_API_KEY` - API 密钥（必填）
- `GEMINI_BASE_URL` - API 端点（可选，默认 `https://generativelanguage.googleapis.com/v1beta`）
- `GEMINI_MODEL` - 直接指定模型（最高优先级）
- `GEMINI_DEFAULT_HAIKU_MODEL` / `GEMINI_DEFAULT_SONNET_MODEL` / `GEMINI_DEFAULT_OPUS_MODEL` - 按能力级别映射
- `GEMINI_DEFAULT_HAIKU_MODEL_NAME` / `DESCRIPTION` / `SUPPORTED_CAPABILITIES` - 显示名称和描述
- `GEMINI_SMALL_FAST_MODEL` - 快速任务使用的模型（可选）

模型映射优先级（`src/services/api/gemini/modelMapping.ts`）：
1. `GEMINI_MODEL` - 直接覆盖
2. `GEMINI_DEFAULT_*_MODEL` - 独立配置（推荐）
3. `ANTHROPIC_DEFAULT_*_MODEL` - 向后兼容 fallback（已废弃）
4. 原样返回 Anthropic 模型名

使用示例：
```bash
export CLAUDE_CODE_USE_GEMINI=1
export GEMINI_API_KEY="your-api-key"
export GEMINI_DEFAULT_SONNET_MODEL="gemini-2.5-flash"
export GEMINI_DEFAULT_OPUS_MODEL="gemini-2.5-pro"
```

### 关键类型文件

- **`src/types/global.d.ts`** — 声明 `MACRO`、`BUILD_TARGET`、`BUILD_ENV` 和内部 Anthropic 专用标识符。
- **`src/types/internal-modules.d.ts`** — `bun:bundle`、`bun:ffi`、`@anthropic-ai/mcpb` 的类型声明。
- **`src/types/message.ts`** — 消息类型层次结构（UserMessage、AssistantMessage、SystemMessage 等）。
- **`src/types/permissions.ts`** — 权限模式和结果类型。

## 测试

- **框架**: `bun:test`（内置断言 + mock）
- **单元测试**: 就近放置于 `src/**/__tests__/`，文件名 `<module>.test.ts`
- **集成测试**: `tests/integration/` — 4 个文件（cli-arguments、context-build、message-pipeline、tool-chain）
- **共享 mock/fixture**: `tests/mocks/`（api-responses、file-system、fixtures/）
- **命名**: `describe("functionName")` + `test("behavior description")`，英文
- **Mock 模式**: 对重依赖模块使用 `mock.module()` + `await import()` 解锁（必须内联在测试文件中，不能从共享 helper 导入）
- **当前状态**: ~1623 个测试 / 114 个文件（110 个单元 + 4 个集成）/ 0 失败（详见 `docs/testing-spec.md`）

## 使用本代码库

### API 配置 (/login)

首次运行后，在 REPL 中输入 `/login` 进入配置界面：
- 选择 **Anthropic Compatible** 对接第三方 API（OpenRouter、AWS Bedrock 代理等）
- 选择 **OpenAI** 或 **Gemini** 使用对应协议
- 字段：Base URL、API Key、Haiku/Sonnet/Opus 模型 ID

支持环境变量方式配置，详见各兼容层章节。

### Teach Me 学习模式

项目内置交互式学习 skill，在 REPL 中输入：
```
/teach-me Claude Code 架构
/teach-me React Ink 终端渲染 --level beginner
/teach-me Tool 系统 --resume
```

- 自动诊断水平，跳过已知、聚焦薄弱
- 学习进度保存在 `.claude/skills/teach-me/`
- 支持 `--resume` 断点续学

### 开发注意事项

- **不要尝试修复所有 tsc 错误** — 它们来自反编译，不影响运行时。
- **功能标志** — 默认全部关闭（`feature()` 返回 `false`）。开发/构建各有自己的默认启用列表。不要在 `cli.tsx` 中重定义 `feature` 函数。
- **React Compiler 输出** — 组件有反编译的记忆化样板代码（`const $ = _c(N)`）。这是正常的。
- **`bun:bundle` import** — `import { feature } from 'bun:bundle'` 是 Bun 内置模块，由运行时/构建器解析。不要用自定义函数替代它。
- **`src/` 路径别名** — tsconfig 将 `src/*` 映射到 `./src/*`。像 `import { ... } from 'src/utils/...'` 这样的导入是有效的。
- **MACRO defines** — 集中管理在 `scripts/defines.ts`。开发模式通过 `bun -d` 注入，构建通过 `Bun.build({ define })` 注入。修改版本号等常量只改这个文件。
- **构建产物兼容 Node.js** — `build.ts` 会自动后处理 `import.meta.require`，产物可直接用 `node dist/cli.js` 运行。
- **Biome 配置** — 大量 lint 规则被关闭（反编译代码不适合严格 lint）。`.tsx` 文件用 120 行宽 + 强制分号；其他文件 80 行宽 + 按需分号。
