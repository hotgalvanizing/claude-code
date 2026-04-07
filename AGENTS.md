# 架构概览 (Architecture Overview)

本文档提供 `claude-code-best` (CCB) 项目的高层架构和开发规范，旨在帮助 AI 助手或开发者快速了解并参与本代码库的开发、重构、测试与调试。

## 1. Project Overview (项目概览)

**目标与定位**：CCB 旨在逆向还原官方 Anthropic Claude Code CLI 工具，提供一个基于终端交互的 AI 编程助手。支持多平台 API（Anthropic、OpenAI、Bedrock 等），并提供丰富的扩展功能（如 Computer Use、Voice Mode、Buddy 等）。

**技术架构**：
- **运行环境与包管理**：`Bun` (>=1.2.0)。
- **项目结构**：Monorepo 架构（通过 Bun Workspaces 支持 `packages/*` 和 `packages/@ant/*` 子包，核心代码位于 `src/`）。
- **核心组件**：
  - UI 渲染层：基于 `React` 和 `Ink`，在终端 (TUI) 中渲染交互界面。
  - 功能集成：Model Context Protocol (MCP) SDK、OpenTelemetry、Sentry 监控、GrowthBook (Feature Flags 管理)。

## 2. Build & Commands (构建与命令)

项目使用 Bun 运行各种开发脚本，关键命令如下：

- **依赖安装**：`bun install`（如遇网络问题，需配置环境变量 `DEFAULT_RELEASE_BASE` 指向镜像代理）。
- **开发运行**：`bun run dev`（会自动挂载默认的特性开关 `Feature Flags` 并启动 REPL）。
- **调试模式**：`bun run dev:inspect`（启动后输出 `ws://localhost...`，使用 VS Code 的 "Attach to Bun" 附加调试）。
- **代码构建**：`bun run build`（将 TS 编译拆分并输出至 `dist/`，主入口为 `dist/cli.js`）。
- **健康检查**：`bun run health`（检测依赖和配置）。
- **文档预览**：`bun run docs:dev`（运行 Mintlify 文档服务）。

## 3. Code Style (代码风格与规范)

项目采用 [Biome](https://biomejs.dev/) 管理代码格式化和 Lint 检查。请遵循以下约定：

- **自动化检查与修复**：
  - 格式化：`bun run format`
  - 代码规范检查：`bun run lint` / `bun run lint:fix`
- **Biome 格式化规则 (`biome.json`)**：
  - 缩进：`2` 个空格。
  - 行宽：普通文件 `80`，`**/*.tsx` 文件 `120`。
  - 引号：单引号 (`single`)。
  - 分号：`.tsx` 文件强制使用 (`always`)，其余类型文件按需使用 (`asNeeded`)。
  - 箭头函数参数括号：按需使用 (`asNeeded`)。
  - 尾随逗号：全局使用 (`all`)。
- **Lint 规则特征**：当前配置禁用了许多严格的规则（如 `noExplicitAny`, `noConsole`, `noUnusedVariables` 等），这意味现有代码包容性较强，但在开发新功能时仍应尽量保持类型安全与代码整洁。

## 4. Testing (测试)

- **框架**：Bun 内置的测试框架 (`bun test`)。
- **运行测试**：使用 `bun test` 执行全部测试。
- **配置与约定**：
  - 全局超时限制设为 `10000ms`（定义于 `bunfig.toml`）。
  - 测试文件放置于对应模块旁的 `__tests__/` 目录内，后缀约定为 `.test.ts`。
  - **开发要求**：修改或重构核心逻辑后，必须运行 `bun test` 确保无回归错误，新功能应包含对应的测试文件。

## 5. Security (安全注意事项)

- **凭证管理**：通过 `/login` 命令对接第三方 API 平台。认证 token 和本地配置存储于系统的用户目录下（如 `~/.claude`），严禁在代码日志或输出中打印完整密钥。
- **遥测与诊断日志**：
  - 项目集成了 Sentry、OpenTelemetry 及 GrowthBook 用于崩溃收集和分析。
  - 编写诊断日志（如 `logForDiagnosticsNoPII`）时，务必**脱敏**，防止意外收集用户代码资产、文件路径或私人信息 (PII)。
- **沙盒与执行**：CLI 工具拥有对本地文件系统的极高权限，在接入如 `Computer Use` 或直接执行 Bash 脚本相关的工具时，务必对不安全的 Payload 或指令进行合理转义（如使用 `shell-quote`）以防止命令注入攻击。

## 6. Configuration (配置与特性管理)

- **本地配置源**：核心运行配置由 `src/utils/config.ts` 及 `src/utils/auth.ts` 读取本地系统环境变量及 `.claude` 目录。
- **Feature Flags (特性开关)**：
  - 项目大量使用特性开关（如 `BUDDY`, `VOICE_MODE`, `BRIDGE_MODE` 等）控制实验性功能的加载。
  - **启用方式**：开发期间可通过设置环境变量 `FEATURE_<FLAG_NAME>=1`（例如 `FEATURE_BUDDY=1 bun run dev`）临时启用特性，`scripts/dev.ts` 会将其解析为 Bun 的 `--feature` 参数。
- **网络代理**：支持自动读取系统的代理配置环境变量（如 `HTTP_PROXY`, `HTTPS_PROXY`），并在请求时自动挂载 `https-proxy-agent`。
