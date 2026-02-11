# OpenClaw 代码架构学习笔记（learn.md）

> 目标：给新读者一条“先能跑通主链路，再分模块深入”的阅读路径。

## 1) 先建立全局心智模型

OpenClaw 可以先理解成三层：

1. **入口与命令层（CLI）**：解析命令、路由到具体子命令与执行逻辑。
2. **控制平面层（Gateway）**：统一 WebSocket/HTTP 服务，承接客户端、节点、渠道和事件流。
3. **能力与扩展层（Channels/Plugins/Agents）**：渠道接入、插件扩展、Agent 运行与会话路由。

如果你只记一句话：**CLI 发起动作，Gateway 编排状态与协议，Channels/Plugins/Agents 完成“连外部世界 + 执行智能体”的闭环。**

---

## 2) 推荐阅读顺序（从“能跑”到“能改”）

### 第 0 步：看文档里的官方架构定义

- `docs/concepts/architecture.md`
- 先看：Gateway 是唯一控制平面、客户端/节点都走同一 WS 协议、事件不回放、握手必须先 `connect`。

### 第 1 步：看程序总入口

- `src/index.ts`
- 重点看：
  - 启动前置：`loadDotEnv`、`normalizeEnv`、`assertSupportedRuntime`。
  - `buildProgram()` 是 CLI 的真正入口。
  - 主模块里统一挂了全局异常处理与 `program.parseAsync`。

### 第 2 步：看 CLI 是怎么“拼装”出来的

- `src/cli/program/build-program.ts`
- `src/cli/program/command-registry.ts`
- `src/cli/program/register.subclis.ts`
- 重点看：
  - `registerProgramCommands` 把命令模块装配进 Commander。
  - `commandRegistry` 里有“快速路由”（例如 health/status/sessions）。
  - `register.subclis.ts` 是大量子 CLI 的懒加载中心（gateway、models、nodes、plugins 等）。

### 第 3 步：看 Gateway 的总装配（核心）

- `src/gateway/server.ts`
- `src/gateway/server.impl.ts`
- 重点看：
  - `startGatewayServer` 是核心装配函数，负责配置校验/迁移、插件加载、通道方法集合、运行时配置解析。
  - WebSocket/HTTP、Control UI、Canvas Host、Tailscale、Cron、Discovery 等都在这里汇总接线。
  - 这层是“最多副作用”区域，改动要特别谨慎。

### 第 4 步：看协议与运行时边界

- `src/gateway/protocol/schema.ts`
- `src/gateway/server-ws-runtime.ts`
- `src/gateway/server-methods.ts`
- 重点看：
  - 协议 schema（TypeBox）定义了请求/响应/事件边界。
  - WS 处理层负责连接生命周期与消息分发。
  - methods 层承接业务动作，是最常见的“加能力”切入点。

### 第 5 步：看“消息去哪个 Agent”

- `src/routing/resolve-route.ts`
- 重点看：
  - 匹配优先级：peer > parentPeer > guild/team > account > channel > default。
  - `sessionKey` 和 `mainSessionKey` 的生成逻辑，直接影响会话隔离、并发和历史归档。

### 第 6 步：看 Agent 命令主流程

- `src/commands/agent.ts`
- 重点看：
  - 参数校验、会话解析、模型与 thinking 配置、技能快照、回写 session store。
  - 这是“用户命令 -> 智能体执行 -> 结果投递”的主链路。

### 第 7 步：看渠道与插件扩展机制

- `src/channels/plugins/index.ts`
- `src/plugins/runtime.ts`
- 重点看：
  - 渠道插件统一注册到运行时 registry，再由 `listChannelPlugins` 聚合和排序。
  - 共享逻辑尽量依赖轻量接口，避免在通用路径引入重依赖。

---

## 3) 核心内容架构（按职责拆）

## A. 入口与命令编排

- 关键目录：`src/index.ts`、`src/cli/**`、`src/commands/**`
- 职责：
  - 启动环境准备（env/runtime/logging）
  - CLI 命令注册、参数处理、命令执行
  - 具体业务命令落在 `src/commands/**`
- 典型修改场景：新增命令、调整参数、优化输出格式

## B. Gateway 控制平面

- 关键目录：`src/gateway/**`
- 职责：
  - 统一 WS/HTTP 入口
  - 连接管理、事件广播、方法分发
  - 配置热加载、服务发现、健康检查、定时任务
- 典型修改场景：新增网关方法、扩展事件、改握手/鉴权策略

## C. 路由与会话

- 关键目录：`src/routing/**`、`src/config/sessions.ts`
- 职责：
  - 从 channel/account/peer 决定 agent
  - 生成稳定 session key，保障多渠道和多会话隔离
- 典型修改场景：多 Agent 路由策略、线程继承策略、会话粒度

## D. Agent 执行与模型策略

- 关键目录：`src/commands/agent.ts`、`src/agents/**`
- 职责：
  - 会话上下文、模型选择、thinking/verbose 级别
  - fallback、技能快照、结果投递
- 典型修改场景：模型优先级、运行超时、技能加载

## E. 渠道与插件生态

- 关键目录：`src/channels/**`、`src/plugins/**`、`extensions/*`
- 职责：
  - 渠道接入统一抽象
  - 插件能力（工具、命令、http handler、channel 等）动态注册
- 典型修改场景：新增渠道、增强插件生命周期、渠道能力对齐

---

## 4) 阅读时要重点关注的“高风险点”

1. **协议兼容性**
   - Gateway 协议一旦改动，会影响 CLI、Web UI、macOS/iOS/Android 节点与插件。
2. **会话键与路由规则**
   - `sessionKey` 规则变化会影响历史、上下文连续性和并发隔离。
3. **配置迁移与自动修复**
   - `server.impl.ts` 在启动阶段会做迁移/自动启用插件，改这里要考虑升级路径与回滚。
4. **插件加载时机**
   - 命令懒加载 + 插件 registry 初始化顺序不当，容易出现“命令可见但能力未注册”。
5. **跨渠道一致性**
   - 改共享逻辑时，务必检查 core channels + extensions 是否行为一致（路由、allowlist、pairing、命令门控）。

---

## 5) 建议的学习节奏（3 天）

- **Day 1：跑通链路**
  - 看 `docs/concepts/architecture.md` + `src/index.ts` + CLI 注册。
  - 目标：知道“一个命令如何进入系统”。
- **Day 2：深挖 Gateway**
  - 读 `server.impl.ts`、protocol schema、ws runtime。
  - 目标：知道“请求如何在控制平面被处理并广播”。
- **Day 3：扩展能力**
  - 读 `resolve-route.ts`、`commands/agent.ts`、channels/plugins。
  - 目标：能独立评估一个功能改动该落在哪层、会影响哪些表面。

---

## 6) 快速索引（最小必读清单）

1. `docs/concepts/architecture.md`
2. `src/index.ts`
3. `src/cli/program/build-program.ts`
4. `src/cli/program/command-registry.ts`
5. `src/cli/program/register.subclis.ts`
6. `src/gateway/server.impl.ts`
7. `src/gateway/protocol/schema.ts`
8. `src/routing/resolve-route.ts`
9. `src/commands/agent.ts`
10. `src/channels/plugins/index.ts`
11. `src/plugins/runtime.ts`

如果你准备做第一个 PR，建议从“新增一个小 CLI 子命令或补一段路由测试”开始，风险最低、反馈最快。
