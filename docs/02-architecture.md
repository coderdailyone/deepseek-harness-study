# 02 · 架构设计解读

> 研究对象：deepseek-harness @ `47f9438`。主要依据：`docs/architecture.md`、`docs/cordis-primer.md`、`docs/capability-seams.md`（生成目录）、`docs/tool-execution-pipeline.md`，以及 `.agents/notes/implemented/architecture/` 下的奠基决策档案。

## 〇、总纲：反常识的取舍

大多数 agent 框架的形状是"一个核心引擎 + 插件挂载点"。dsh 走到极端：**没有特权核心可改**。模型适配器、工具注册表、会话日志、持久化、沙箱、审批、甚至 agent loop 本身，全是插件。`agent-loop` 的自我定位（capability-seams 目录原文）："the one concrete loop plugin——扩展包只依赖 `dsh-agent` 的事件和服务，不依赖 loop 包本身"。**连循环都可整体换掉**，其他一切只认它背后的事件契约。

这不是审美，是需求推出来的：MVP 需求即要求 hooks、/goal、compaction、沙箱、权限、UI、MCP、skills 全部能以插件形式添加而不改核心（microkernel 决策档案的 Problem 节）。

## 一、地基：Cordis 五个概念

一切建立在 vendored 的 Cordis 插件框架上：

1. **插件**是实现 Service 的对象（函数带 `inject` + `apply(ctx)`，或 Service 子类）
2. **Context 是服务仓库**：服务认领稳定的 `ctx.<key>`（`ctx.tools`/`ctx.llm`/`ctx.sessions`），他人按 key 找服务而非 import 具体实现
3. **依赖用 `inject` 声明**：插件等声明的服务出现才挂载——启动顺序不用手排，由依赖关系涌现
4. **通信用 typed events**（TypeScript declaration merging 声明），四种分发模式各有铁律用途
5. **注册皆效果（effect）**：任何注册走 `ctx.effect()`/`ctx.on()` 且返回销毁器——热重载和卸载白送

分发模式的选择本身就是架构决策：

| 模式 | 语义 | 承担什么 |
|---|---|---|
| `waterfall` | 环绕中间件：`next()` 委托，不调则短路 | 一切"插件可改写/拦截/兜底"的点：`agent/pre-step`、`agent/request`、`tools/pre-execute` 等 |
| `serial` | 按注册序 await | 有序检查点：`agent/turn-stopping` |
| `parallel` | 全员并发 await | 每个监听者必须独立得到机会的点：`session/flush` 落盘检查点 |
| `emit` | 同步 fire-and-forget | 纯通知：inbox 变化、生命周期、错误 |

被拒绝的替代方案（microkernel 决策档案）：自造 koa-compose 式中间件栈、显式相位状态机——都会重新发明 Cordis 已有的分发/销毁/重载语义；作为 Cordis effect 的监听器天然获得 HMR 与销毁。

## 二、支柱一：事件溯源会话（log 即状态）

`.agents/notes/implemented/architecture/2026-06-11-event-sourced-sessions.md`：

- **`Session` 是 append-only 的类型化事件日志，是唯一事实源**；模型消息历史从日志投影（`deriveMessages()`）
- 被拒绝的替代方案一针见血："可变消息数组 + 事件通知——更简单，但状态和日志会漂移；事件溯源下**日志就是状态，漂移在结构上不可能**"
- 推论即军规 **model-visible ⟺ logged**：任何进入模型请求的内容必须能从日志重建，运行时断言兜底；新的模型可见输入 = 新的 session 事件类型
- 彻底程度：连收件箱都是日志投影（`agent/inbox/spliced` 事件先落日志再动内存，跨重启不丢未处理输入）；连请求配置（provider/model/system prompt/工具集）都以 `request/header` 事件入日志，配置变更全程可审计

**性能配套**（`2026-06-14-session-persistence.md`）：追加同步（热路径不碰 I/O）、持久化插件 write-behind、每 turn 末的 `session/flush`（parallel 模式）检查点排空。

**崩溃哲学**：崩掉的 turn 补写关闭事件（未答复的 assistant 调用补风险分级的错误结果、缺失的 `step/end`、`turn/end {interrupted}`），**绝不截断**——"一个被打断的长自主运行可能包含大量有效工作，截断是静默销毁"。合成结果同时保证恢复后的 provider transcript 合法。

## 三、支柱二：微内核事件分类学（loop 的扩展点即事件）

一步（step）= 一次模型请求 + 它调的工具；一轮（turn）= 零或多步。循环的可插拔点全部是事件：

```
turn/start
  认领 inbox 输入 → 拼装 prompt 段落 + 工具 schema
  agent/pre-step (waterfall)     ← 插件可改写/拒绝模型将看到的内容
    step/start
    从日志投影模型历史
    agent/request (waterfall)    ← 插件可换模型/改参数
    llm/stream → assistant/chunk* → assistant/message
    tool/call* → 工具管线（见下） → tool/result*
    step/end
  agent/turn-stopping (serial)   ← 有序的"还有事吗"检查点
turn/end
```

`turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*` 是持久 session 事件；其余是三个域的活扩展点（session 事件 = 持久事实、`agent/*` 事件 = 在途拦截、capability 事件 = 策略接缝）。

**工具执行管线**（`docs/tool-execution-pipeline.md`）是同一思想的纵向切面，一次调用穿过：

1. `tools/pre-execute` waterfall（hooks、权限、沙箱策略）
2. **单调守卫**（只许 deny 或弃权，身份受保护——防插件秩序问题）
3. `ctx.approval` 一次性人工审批（**缺席即 fail-closed 拒绝**）
4. `tools/execute` waterfall（超时/重试/指标，环绕真正执行体）
5. `tools/post-execute` waterfall（接受/拦截/替换/补上下文）
6. 归一化（任何抛出变 isError 结果）→ `finalizeContent` → `tools/result` 冻结通知

Claude Code/Codex 的 hook 桥、沙箱、审批全是这条管线上的普通监听者——"加能力不改 loop"的实证。

## 四、支柱三：能力接缝（seam）三角色分包

`2026-06-13-capability-seams.md`，最有辨识度的包组织学。一个可换能力 = 三个角色：

| 角色 | 职责 | bash 范例 |
|---|---|---|
| **Service Definition** | 拥有 `ctx.<key>` 与词表类型的抽象 Service | `dsh-shell` |
| **Service Provider** | 实现（可多个并存） | `dsh-bash-local` / `dsh-bash-sandbox` |
| **Consumer** | 模型和插件编程的对象（工具 schema） | `dsh-tool-bash` |

**Why**：三个关注点变化速率不同、原因不同。捆在一个包里，换个沙箱执行器就会震动模型看到的工具 schema——尽管模型面契约根本没变。分开后"一次 provider 替换改变整个产品"：文件系统和子进程 provider 共享一个执行世界，把它们指到远程沙箱（E2B），Bash、PTY、LSP 全体跟着搬走，零 fork。

生成的 seam 目录显示规模：**60+ 个 `ctx.*` 服务**。亮点：

- `ctx.subagents` 六个 provider 并列：进程内 spawn/fork、ACP、Codex、Claude Code、dsh-sdk——"子代理"从新起子 agent 到**委托给另一个产品的一轮对话**都在同一接口后
- `ctx.web` 四个 provider：Exa/Perplexity/DeepSeek 搜索 + HTTP 抓取
- `ctx.credentials`："配置携带秘密的*引用*，provider 拥有值，消费者每次操作现解析——轮换的凭据下一个请求就生效"
- 克制条款："**不要预防性分包**——只有一个可想象 provider 的能力保持单包，直到第二个出现"

## 五、组合系统：profile / bundle / patch

运行中的 dsh = boot 时按序叠层组合出来的插件树：

```
dsh-base bundle（模型适配器/工具/持久化/沙箱/审批/设置/凭据/遥测）
  → dsh-web-app 或 dsh-headless bundle
    → profile 的 cordis.patch.yml
      → 用户 home 层 patch
        → --patch 命令行覆盖层
```

patch 按 id 定位任意行、整体替换其 config 或插入新行。`dsh --profile web --dump-config` 打出机器实际启动的树——**打出的每一行都可被你自己的 patch 替换**。实测 headless profile 约 60+ 行，连 plan-mode 的 system prompt 原文、权限三档 preset（read-only / workspace-write / danger-full-access）、遥测开关（默认 DISABLED）全躺在可 patch 的配置里。

## 六、架构与 agent 开发方法互相成就

这个架构本身就是为 agent 大规模开发优化的：

- **类型化事件 + declaration merging** → 事件词表机器可查，生成目录可对照声明与分发点（`@mode` 标签校验）
- **seam 三角色** → agent 改 provider 绝不碰坏模型面契约，爆炸半径被包结构锁死
- **注册皆效果** → agent 写的插件天然可卸载可重载
- **一切从日志投影** → 回放测试（keyless snapshot）不需要 mock 整个世界，日志即完整现场
- **60+ 服务目录、模块图、配置目录全部生成 + 新鲜度门禁** → agent 的"地图"永不过期
