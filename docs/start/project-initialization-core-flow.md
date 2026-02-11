---
summary: "从安装依赖到 CLI 入口执行的核心启动链路（面向开发者）"
read_when:
  - 你第一次在本地拉起 OpenClaw 项目
  - 你需要快速定位 CLI 启动链路和关键代码入口
  - 你要排查“为什么本地命令没有按预期执行”
title: "项目初始化与核心代码运行流程"
sidebarTitle: "初始化与运行流程"
---

# 项目初始化与核心代码运行流程

这份文档按“**首次拉起仓库** → **运行命令** → **进入核心代码**”的顺序，梳理 OpenClaw 的主线流程，方便你快速建立全局认知。

## 1) 环境与前置要求

### 运行时版本

- Node.js 需要 `22.12.0+`（`package.json` 的 `engines.node`）。
- 包管理器使用 `pnpm`（仓库声明 `pnpm@10.23.0`）。

### 推荐准备

在仓库根目录执行：

```bash
pnpm install
prek install
```

说明：

- `pnpm install` 安装项目依赖。
- `prek install` 安装/启用与 CI 对齐的本地 pre-commit 流程（仓库指南推荐）。

## 2) 开发常用入口命令

OpenClaw 日常开发常用这几条：

```bash
pnpm openclaw ...
pnpm dev
pnpm build
pnpm test
pnpm check
```

你可以这样理解：

- `pnpm openclaw ...`：最常用，本地开发态执行 CLI。
- `pnpm dev`：等价于通过 Node 入口脚本拉起 CLI。
- `pnpm build`：构建 `dist/*` 产物。
- `pnpm test`：运行测试。
- `pnpm check`：格式、类型检查和 lint 的组合检查。

## 3) 从命令到代码：主执行链路

下面是最关键的链路（以 `pnpm openclaw` 为例）：

1. `package.json` 的 `openclaw` script 指向 `node scripts/run-node.mjs`。
2. `scripts/run-node.mjs` 检查 `dist` 是否过期：
   - 若过期，先触发 `pnpm exec tsdown --no-clean`。
   - 构建完成写入 `dist/.buildstamp`，然后继续执行。
3. 脚本最终会调用 Node 执行 `openclaw.mjs`。
4. `openclaw.mjs` 负责加载构建产物入口：
   - 优先尝试 `./dist/entry.js`
   - 其次尝试 `./dist/entry.mjs`
5. `src/entry.ts`（编译后到 `dist/entry.*`）中完成基础初始化：
   - 安装 warning filter、环境变量规范化。
   - 必要时通过 respawn 注入 `--disable-warning=ExperimentalWarning`。
   - 解析 `--profile` 等早期参数。
   - 最终动态导入 `./cli/run-main.js` 并调用 `runCli`。
6. `src/cli/run-main.ts` 的 `runCli` 进入 Commander 主流程：
   - 加载 `.env`、运行时校验、PATH 处理。
   - 先尝试轻量路由（`tryRouteCli`）。
   - 构建 `program`，注册子命令/插件命令。
   - 执行 `program.parseAsync(...)` 真正分发到对应命令实现。

> 一句话记忆：
> **pnpm script → run-node（按需构建）→ openclaw.mjs（加载 dist）→ entry（初始化）→ runCli（命令分发）**。

## 4) 初始化阶段的关键行为

### 4.1 构建是否触发（`scripts/run-node.mjs`）

该脚本会比较以下时间戳决定是否重建：

- `dist/.buildstamp`
- `dist/entry.js`
- `tsconfig.json`、`package.json`
- `src/` 下源码（会跳过 `.test.ts` 等测试文件）

这让本地开发时可以避免每次都完整构建，提升启动速度。

### 4.2 入口兜底（`openclaw.mjs`）

`openclaw.mjs` 同时兼容 `dist/entry.js` 和 `dist/entry.mjs`，若都不存在会直接抛错，提示构建产物缺失。

### 4.3 CLI 预处理（`src/entry.ts`）

此阶段做的事情包括：

- 将 `--no-color` 映射到 `NO_COLOR=1/FORCE_COLOR=0`。
- Windows 场景下做 `argv` 清洗（防止 `node.exe` 残留参数影响命令解析）。
- 处理 `--profile`，将 profile 注入环境，保证后续命令一致读取。

### 4.4 命令注册与执行（`src/cli/run-main.ts`）

`runCli` 里有几个容易忽略的点：

- 先做运行时校验（不符合最低 Node 版本会提前失败）。
- help/version 场景会跳过某些高开销注册路径。
- 插件 CLI 命令会在 parse 前注册进 Commander。

## 5) 推荐的本地初始化检查清单

第一次拉起后，建议至少跑一次：

```bash
pnpm build
pnpm test
pnpm check
```

如果你只想快速确认 CLI 能跑通：

```bash
pnpm openclaw --help
```

## 6) 故障排查（按执行链路定位）

- **`pnpm openclaw` 一上来就失败**：先看 `scripts/run-node.mjs` 的构建阶段是否报错。
- **提示缺少 `dist/entry.*`**：说明构建产物未生成或被清理，先执行 `pnpm build`。
- **参数行为异常**：优先检查 `src/entry.ts` 的参数预处理（尤其 Windows 与 profile 逻辑）。
- **子命令找不到**：检查 `src/cli/run-main.ts` 的注册与 `parseAsync` 前逻辑。

## 7) 延伸阅读

- [开发环境与调试建议](/help/environment)
- [测试说明](/help/testing)
- [文档目录索引](/start/docs-directory)
