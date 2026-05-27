# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 语言规范

始终使用简体中文进行对话、编写文档和代码注释。

## 构建命令

```bash
npm ci                          # 安装依赖
npm run build                   # 完整构建：lint → playwright bundle → CRX 库 → examples → tests
npm run build:crx               # 仅构建 playwright-crx 库到 lib/
npm run build:examples           # 构建 examples（recorder-crx 和 todomvc-crx）
npm run build:examples:recorder  # 仅构建 recorder-crx 扩展
npm run build:tests              # 构建 test 扩展
npm run lint                    # ESLint 检查
npm test                        # 运行测试（需要在 tests 目录下先执行 npm run test:install 安装 chromium）
npm run test-ui                 # Playwright UI 模式运行测试
```

## 核心架构

playwright-crx 将 Playwright 封装为 Chrome Extension，通过 `chrome.debugger` API 实现 CDP 协议通信，无需启动独立浏览器进程。

### 进程内桥接 (`src/index.ts`)

项目在单个 Service Worker 进程中同时运行 Playwright 的 server 端和 client 端，通过 `DispatcherConnection` ↔ `CrxConnection` 桥接：

```
client 端 (ChannelOwner)  ←→  dispatcher 端 (SdkObject)
```

- **client 端** (`src/client/`)：实现 `Crx`/`CrxApplication` 等 ChannelOwner，对外暴露 Playwright 风格的 API
- **dispatcher 端** (`src/server/`)：实现实际的浏览器控制、录制、回放逻辑
- `src/protocol/`：定义 CRX 特定的 channel 类型和参数校验

### 传输层 (`src/server/transport/`)

`CrxTransport` 替代 Playwright 的 WebSocket/Pipe 传输，使用 `chrome.debugger.attach()` 和 `chrome.debugger.sendCommand()` 与浏览器 CDP 通信。

### 关键类

| 类 | 文件 | 职责 |
|---|---|---|
| `Crx` (server) | `src/server/crx.ts` | 管理 CRBrowser 创建、CrxApplication 生命周期、incognito 支持 |
| `CrxApplication` | `src/server/crx.ts` | 管理单个 browser context，tab attach/detach，recorder 显示 |
| `CrxRecorderApp` | `src/server/recorder/crxRecorderApp.ts` | 录制器应用：窗口管理、事件分发、代码编辑 |
| `CrxPlayer` | `src/server/recorder/crxPlayer.ts` | 回放录制的操作 |
| `Crx` (client) | `src/client/crx.ts` | 对外暴露的 public API（`crx.start()`, `crxApp.attach()` 等） |

### Recorder 窗口模式

两种 UI 模式：
- **Side Panel**：`SidepanelRecorderWindow`，通过 `chrome.sidePanel` API 在标签页侧边显示，默认启用，可通过 `tabId` 绑定到单个标签页
- **Popup**：`PopupRecorderWindow`，通过 `chrome.windows.create({ type: 'popup' })` 创建独立窗口

用户可在扩展选项页中切换。

### Shims (`src/shims/`)

为在 Chrome Extension 环境中运行的 Node.js 代码（Playwright 本身）提供浏览器端 shim，如 `fs`、`crypto`、`net` 等模块的浏览器实现。通过 `vite.config.mts` 中的 alias 机制注入。

### Examples

- `examples/recorder-crx/`：Playwright 录制器 Chrome 扩展，入口为 `src/background.ts` (Service Worker)
- `examples/todomvc-crx/`：使用 playwright-crx 作为库的示例扩展

### Playwright 源码

`playwright/` 目录是上游 Playwright 的 git subtree，通过以下命令更新：

```bash
git subtree pull --prefix=playwright git@github.com:microsoft/playwright.git v1.XX.0 --squash
```

### 测试

## 更新记录

每次修改代码后，必须在 `docs/更新记录.md` 中同步更新变更说明。在该文件的「未发布」分类下添加条目，版本发布时迁移到对应的版本号下。

---

测试位于 `tests/`，使用 Playwright Test 框架测试扩展本身：
- `tests/crx/`：CRX API 测试
- `tests/server/`：服务端单元测试
- `tests/test-extension/`：测试用的 Chrome 扩展
- `tests/playwright.config.ts`：测试配置，启动 chromium 并加载扩展
