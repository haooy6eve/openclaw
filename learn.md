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

---

## 8) 引用代码逐文件拆解（每一步做什么）

> 这部分按“从入口到执行”的顺序，把上面引用的关键代码文件逐个拆开，方便你对照源码逐行读。

### 8.1 `src/index.ts`（CLI 启动入口）

1. **加载环境和运行前置**：先做 `.env` 加载、环境归一化、PATH 修正、日志捕获和运行时版本校验。
2. **构建命令程序**：调用 `buildProgram()` 获取 commander 程序对象。
3. **判断是否主模块运行**：如果是被直接执行，就注册全局异常处理并 `parseAsync(process.argv)`。
4. **导出复用函数**：把若干工具函数 re-export，供其他模块或测试复用。

### 8.2 `src/cli/program/build-program.ts`（CLI 装配总控）

1. 创建 `Command` 实例。
2. 创建程序上下文（版本信息、通用参数上下文）。
3. 配置帮助文案和 pre-action hooks。
4. 调用命令注册器注册所有主命令和子命令。
5. 返回完整 program。

### 8.3 `src/cli/program/command-registry.ts`（命令注册与快速路由）

1. 定义 `CommandRegistration` 数据结构，把“注册行为”和“可选路由行为”绑定。
2. 声明快速路由（如 health/status/sessions/memory），减少不必要的全量加载。
3. 在 `commandRegistry` 列表里集中维护各模块命令的注册入口。
4. `registerProgramCommands` 遍历 registry 完成装配。
5. `findRoutedCommand` 负责在 argv path 上查找命中的路由处理器。

### 8.4 `src/cli/program/register.subclis.ts`（子 CLI 懒加载中心）

1. 定义 `entries`：每个子 CLI 的名称、描述、注册函数。
2. 每个 `register` 都通过动态 `import()` 延迟加载真实模块。
3. 对 pairing/plugins 等依赖插件注册表的命令，先初始化 plugin CLI 再注册命令。
4. 提供 `registerSubCliByName` 支持按需注册单个子命令。
5. 根据环境变量决定是否 eager 注册，兼顾启动速度与调试可见性。

### 8.5 `src/gateway/server.impl.ts`（Gateway 主装配实现）

1. **启动前配置处理**：读取配置快照、执行 legacy 迁移、校验非法配置。
2. **插件与渠道初始化**：加载插件、聚合 channel methods、创建各子系统 logger。
3. **运行时参数解析**：绑定地址、鉴权、control UI、tailscale、http endpoint 开关。
4. **服务组件接线**：WS/HTTP server、canvas host、hooks、cron、discovery、health、维护定时器。
5. **生命周期管理**：处理关闭逻辑、重载逻辑、会话与节点状态广播。

### 8.6 `src/gateway/server-methods-list.ts`（Gateway 方法清单）

1. 统一维护“可调用 Gateway 方法”的集合。
2. 将 core methods 与扩展 methods 合并成最终列表。
3. 给协议暴露层/日志层/分发层提供一致的方法名来源。

### 8.7 `src/gateway/protocol/schema.ts`（协议 Schema 边界）

1. 使用 TypeBox 定义 request/response/event 数据结构。
2. 约束连接握手、方法参数、返回负载的结构。
3. 为 runtime 校验和下游代码生成（如 schema/swift 模型）提供源定义。

### 8.8 `src/routing/resolve-route.ts`（路由决策）

1. 对 channel/account/peer/guild/team 做标准化。
2. 从配置中筛选可匹配 binding。
3. 按固定优先级逐层匹配（peer → parentPeer → guild/team → account/channel）。
4. 选择 agent 后生成 `sessionKey` 与 `mainSessionKey`。
5. 返回 `matchedBy` 便于调试与可观测。

### 8.9 `src/routing/session-key.ts`（会话键规则）

1. 统一约定主会话键和 peer 会话键格式。
2. 根据 dm scope/account/channel/peer 组合生成可复现键。
3. 提供 agent id 归一化与 key 构造辅助，确保跨渠道一致。

### 8.10 `src/commands/agent.ts`（Agent 命令执行主线）

1. 做输入校验（message、session 目标、agent id）。
2. 解析会话（session id/key/store）并加载已有状态。
3. 解析模型、thinking、verbose、timeout 等执行参数。
4. 处理技能快照与会话回写（必要时写回 session store）。
5. 执行 agent（含 fallback）并按策略投递结果。

### 8.11 `src/channels/plugins/index.ts`（渠道插件聚合）

1. 从 active plugin registry 读取所有 channel 插件。
2. 对重复 channel id 去重。
3. 按内置顺序 + 插件自定义顺序排序。
4. 对外暴露 `listChannelPlugins/getChannelPlugin/normalizeChannelId` 等接口。

### 8.12 `src/plugins/runtime.ts`（插件运行时单例状态）

1. 初始化全局 registry state（挂在 `globalThis`）。
2. 提供 set/get/require 三类访问器。
3. 允许在不同加载路径中共享同一份运行时插件注册表。

### 8.13 `src/plugins/registry.ts`（插件注册表结构定义）

1. 定义插件注册表的数据模型（plugins/tools/hooks/channels/providers 等）。
2. 规定插件加载结果如何被聚合。
3. 为 runtime 和 loader 提供统一类型契约。

### 8.14 `docs/concepts/architecture.md`（架构文档基线）

1. 提供 Gateway 架构总览和连接生命周期。
2. 说明客户端、节点、WebChat 等角色边界。
3. 给出握手协议、远程访问和系统不变量，是“代码阅读前的语义底座”。
