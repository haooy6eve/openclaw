# OpenClaw 代码架构学习笔记（learn.md）

> 目标：给新读者一条“先跑通主链路，再分模块深入”的阅读路径；也可在代码更新后用作快速复查清单。

## 1) 先建立全局心智模型

OpenClaw 可以先理解成三层：

1. **入口与命令层（CLI）**：解析命令、装配子命令、执行具体业务。
2. **控制平面层（Gateway）**：统一 WebSocket/HTTP 服务，管理连接、事件、方法调用。
3. **能力与扩展层（Channels / Plugins / Agents）**：渠道接入、插件扩展、Agent 执行和会话路由。

一句话总结：**CLI 发起动作，Gateway 编排协议与状态，Channels/Plugins/Agents 完成“连接外部世界 + 执行智能体”闭环。**

---

## 2) 推荐阅读顺序（从“能跑”到“能改”）

### 第 0 步：官方架构定义

- `docs/concepts/architecture.md`
- 先确认核心约束：Gateway 是单一控制平面；客户端/节点统一走 WS；首帧必须 `connect`；事件默认不回放。

### 第 1 步：程序总入口

- `src/index.ts`
- 重点看：
  - 启动前置：`loadDotEnv`、`normalizeEnv`、`assertSupportedRuntime`。
  - `buildProgram()` 如何进入 CLI 主流程。
  - 主模块下的全局异常与 `program.parseAsync` 收敛。

### 第 2 步：CLI 装配与懒加载

- `src/cli/program/build-program.ts`
- `src/cli/program/command-registry.ts`
- `src/cli/program/register.subclis.ts`
- 重点看：
  - `registerProgramCommands` 如何集中注册命令。
  - `commandRegistry` 的快速路由（health/status/sessions/memory）。
  - `register.subclis.ts` 的子 CLI 懒加载（gateway/models/nodes/plugins/pairing 等）。

### 第 3 步：Gateway 总装配（核心）

- `src/gateway/server.ts`
- `src/gateway/server.impl.ts`
- 重点看：
  - `startGatewayServer`：配置校验/迁移、插件装载、运行时配置解析。
  - WS/HTTP、Control UI、Canvas Host、Tailscale、Cron、Discovery 等接线点。
  - 这层副作用最多，任何改动都要评估启动路径和兼容面。

### 第 4 步：协议边界与方法分发

- `src/gateway/protocol/schema.ts`
- `src/gateway/server-ws-runtime.ts`
- `src/gateway/server-methods.ts`
- `src/gateway/server-methods-list.ts`
- 重点看：
  - TypeBox schema 如何定义请求/响应/事件边界。
  - WS 生命周期与消息分发位置。
  - 方法注册清单（包含 core + plugin + channel 方法）如何汇总。

### 第 5 步：路由与会话键

- `src/routing/resolve-route.ts`
- `src/routing/session-key.ts`
- 重点看：
  - 匹配优先级：peer > parentPeer > guild/team > account > channel > default。
  - `sessionKey` / `mainSessionKey` 生成策略对上下文连续性和隔离的影响。

### 第 6 步：Agent 命令主链路

- `src/commands/agent.ts`
- `src/commands/agent/*`
- 重点看：
  - 参数校验、会话解析、模型与 thinking/verbose 策略。
  - 技能快照、session store 回写、结果投递。

### 第 7 步：渠道与插件运行时

- `src/channels/plugins/index.ts`
- `src/channels/registry.ts`
- `src/plugins/runtime.ts`
- `src/plugins/registry.ts`
- 重点看：
  - 渠道插件如何注册、排序、归一化。
  - plugin registry 如何作为运行时单例被读取和注入。

---

## 3) 核心内容架构（按职责拆）

### A. 入口与命令编排

- 关键目录：`src/index.ts`、`src/cli/**`、`src/commands/**`
- 职责：
  - 启动环境准备（env/runtime/logging）
  - CLI 命令注册、参数处理、命令执行
  - 业务命令落在 `src/commands/**`
- 常见改动：新增命令、扩展参数、输出格式调整

### B. Gateway 控制平面

- 关键目录：`src/gateway/**`
- 职责：
  - 统一 WS/HTTP 入口
  - 连接管理、事件广播、方法分发
  - 配置热加载、健康状态、定时任务、服务发现
- 常见改动：新增网关方法、事件扩展、鉴权/握手策略

### C. 路由与会话

- 关键目录：`src/routing/**`、`src/config/sessions.ts`
- 职责：
  - 从 channel/account/peer 解析 agent
  - 构造稳定 session key，保证多渠道/多会话隔离
- 常见改动：多 Agent 路由策略、线程继承、会话粒度

### D. Agent 执行与模型策略

- 关键目录：`src/commands/agent.ts`、`src/agents/**`
- 职责：
  - 模型选择、fallback、thinking 策略
  - 会话上下文与技能快照管理
- 常见改动：模型优先级、超时策略、技能加载规则

### E. 渠道与插件生态

- 关键目录：`src/channels/**`、`src/plugins/**`、`extensions/*`
- 职责：
  - 渠道统一抽象
  - 插件能力（tool/command/http/channel）动态注册
- 常见改动：新增渠道、插件生命周期增强、渠道行为对齐

---

## 4) 阅读时重点关注的高风险点

1. **协议兼容性**
   - 协议改动会同时影响 CLI、Web UI、macOS/iOS/Android 节点和插件调用。
2. **会话键与路由规则**
   - `sessionKey` 规则变化会影响历史、上下文延续和并发隔离。
3. **配置迁移与自动修复**
   - 启动阶段涉及迁移/自动启用插件，改动要覆盖升级路径与失败回滚。
4. **插件加载时机**
   - 命令懒加载和 plugin registry 初始化顺序不当，会出现“命令可见但能力缺失”。
5. **跨渠道一致性**
   - 共享逻辑改动要同步核对 built-in channels + extensions（路由、allowlist、pairing、命令门控）。

---

## 5) 代码更新后：优先复查清单

当你看到“最近代码更新了”，建议按下面顺序快速核对：

1. **入口/注册有没有变**
   - `src/index.ts`
   - `src/cli/program/command-registry.ts`
   - `src/cli/program/register.subclis.ts`
2. **Gateway 方法表和协议有没有变**
   - `src/gateway/server-methods-list.ts`
   - `src/gateway/protocol/schema.ts`
3. **路由与会话规则有没有变**
   - `src/routing/resolve-route.ts`
   - `src/routing/session-key.ts`
4. **插件/渠道注册点有没有变**
   - `src/channels/plugins/index.ts`
   - `src/plugins/runtime.ts`
5. **最后再看文档是否需要同步**
   - `docs/concepts/architecture.md`

> 经验：如果第 2、3 步有变化，`learn.md` 几乎总是需要更新。

---

## 6) 建议学习节奏（3 天）

- **Day 1：跑通链路**
  - 看架构文档 + `src/index.ts` + CLI 注册。
  - 目标：理解“一个命令怎样进入系统”。
- **Day 2：深挖 Gateway**
  - 读 `server.impl.ts` + protocol + ws runtime + methods list。
  - 目标：理解“请求如何被处理并广播”。
- **Day 3：扩展能力**
  - 读 routing/session-key + agent + channels/plugins。
  - 目标：能判断功能该落在哪层、影响哪些面。

---

## 7) 最小必读清单（快速索引）

1. `docs/concepts/architecture.md`
2. `src/index.ts`
3. `src/cli/program/build-program.ts`
4. `src/cli/program/command-registry.ts`
5. `src/cli/program/register.subclis.ts`
6. `src/gateway/server.impl.ts`
7. `src/gateway/server-methods-list.ts`
8. `src/gateway/protocol/schema.ts`
9. `src/routing/resolve-route.ts`
10. `src/routing/session-key.ts`
11. `src/commands/agent.ts`
12. `src/channels/plugins/index.ts`
13. `src/plugins/runtime.ts`

如果你准备做第一个 PR，建议从“补一段路由测试或新增一个小 CLI 子命令”开始，风险低、反馈快。
