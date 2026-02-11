---
summary: "OpenClaw 仓库目录结构与模块职责总览"
read_when:
  - 你刚接手仓库，想先建立目录级地图
  - 你要定位某类功能应该改在哪个目录
  - 你要评估改动影响到哪些子系统
title: "项目结构目录介绍"
sidebarTitle: "项目结构目录"
---

# 项目结构目录介绍

本文从“**顶层目录** → **核心源码目录** → **扩展与应用目录** → **构建与测试目录**”四层来介绍 OpenClaw 的仓库结构。

## 1) 顶层目录（Root）

常见且高频的根目录文件/目录：

- `src/`：核心 TypeScript 源码。
- `docs/`：文档（Mintlify）。
- `extensions/`：扩展插件（独立 workspace package）。
- `apps/`：平台客户端（如 iOS / Android / macOS）。
- `scripts/`：构建、测试、发布、运维等脚本。
- `dist/`：构建产物目录（通常由构建流程生成）。
- `ui/`：Web UI 相关工程。
- `test/`：补充测试资源/测试支撑文件。
- `openclaw.mjs`：CLI 运行入口加载器。
- `package.json`：脚本、依赖、引擎版本、发布配置。

## 2) `src/` 核心源码分层

根据仓库约定，`src/` 里最关键的分区如下：

### 2.1 CLI 与命令

- `src/cli`：CLI 主体 wiring（参数解析、命令注册、入口协同）。
- `src/commands`：各命令业务实现。

你可以把它理解为：

- `src/cli` 决定“怎么进来、怎么分发”；
- `src/commands` 决定“每条命令具体做什么”。

### 2.2 渠道与路由

- `src/channels`：渠道抽象与共享能力。
- `src/routing`：消息路由策略。
- 各渠道目录：
  - `src/telegram`
  - `src/discord`
  - `src/slack`
  - `src/signal`
  - `src/imessage`
  - `src/web`（WhatsApp Web）

这部分是 OpenClaw 的“多渠道消息接入层”。

### 2.3 基础设施与运行时

- `src/infra`：环境变量、错误处理、运行时守卫、通用基础设施。
- `src/media`：媒体处理管线。
- `src/provider-web.ts`：Web provider 相关入口实现。

## 3) `extensions/` 插件扩展层

`extensions/*` 下是插件/扩展包，例如 Teams、Matrix、Zalo、Voice Call 等。

关键约束：

- 插件运行时依赖应放在各自 extension 的 `dependencies`。
- 不要把插件专用依赖随意加到根 `package.json`。
- 避免在插件 `dependencies` 使用 `workspace:*`（npm install 兼容性问题）。

简化理解：

- `src/` 是 core；
- `extensions/` 是可插拔能力；
- 两者通过插件 SDK 和命令/路由机制协同。

## 4) `docs/` 文档体系

`docs/` 是 Mintlify 文档根目录，按主题分区：

- `docs/start`：上手、引导、开发初始化相关文档。
- `docs/channels`：渠道接入说明。
- `docs/gateway`：网关架构、配置、运维。
- `docs/tools`：工具能力与使用。
- `docs/help`：FAQ、调试、提交流程。
- `docs/reference`：命令/API/模板参考。

注意：

- `docs/zh-CN/**` 为生成目录，默认不直接手工修改。

## 5) `apps/` 多端应用层

`apps/` 下通常包含：

- `apps/macos`：macOS 应用工程。
- `apps/ios`：iOS 应用工程。
- `apps/android`：Android 应用工程。
- `apps/shared`：跨端共享代码（如 OpenClawKit）。

这层主要负责用户端体验和设备侧接入。

## 6) `scripts/` 自动化脚本层

`scripts/` 里集中管理构建/测试/运维辅助脚本，例如：

- `scripts/run-node.mjs`：开发态运行入口（可按需触发构建）。
- `scripts/test-*.sh|mjs`：测试与 E2E 脚本。
- `scripts/package-mac-app.sh`：macOS 打包流程脚本。

建议：

- 优先走已有脚本，不要重复手写同类流程。

## 7) 快速定位指南（按需求找目录）

- **改 CLI 参数/命令分发**：先看 `src/cli`。
- **改具体命令行为**：先看 `src/commands`。
- **改某个渠道接入逻辑**：先看对应 `src/<channel>`，再看 `src/channels`/`src/routing`。
- **改文档**：看 `docs/<topic>`。
- **改插件**：看 `extensions/<plugin>`。
- **改移动端/桌面端 App**：看 `apps/<platform>`。

## 8) 建议阅读顺序（新同学）

1. 先看本文建立目录感知。
2. 再看 [项目初始化与核心代码运行流程](/start/project-initialization-core-flow)。
3. 然后看 [开发环境](/help/environment) 和 [测试说明](/help/testing)。
4. 最后进入你要修改的具体子系统目录做深挖。
