# 06 · Subagent 接缝：六个 provider 藏在一个接口后

> 研究对象：deepseek-harness @ `47f9438`。依据：`packages/subagent/subagent/README.md`（154 行的 seam 契约文档，本篇主要来源）、`.agents/notes/implemented/feature/2026-08-04-claude-code-and-codex-subagent-backends.md`、`2026-08-12-production-dsh-excludes-product-subagent-providers.md` 等 20+ 篇 subagent 决策档案。

## 全景：一个注册表，六种"子代理"

`ctx.subagents` 是命名 provider 注册表。委托方用同一套服务 API，provider 决定孩子跑在哪：

| provider | 孩子在哪运行 | 继承父上下文？ |
|---|---|---|
| `subagent-spawn-in-process` | 本进程，全新子 agent | 否 |
| `subagent-fork-in-process` | 本进程，fork 父会话历史 | **是**（看到父的已完成对话） |
| `subagent-acp` | 外进程，ACP 协议 | 否 |
| `subagent-codex` | 外进程，**真 Codex 产品**（`codex app-server --stdio`） | 否 |
| `subagent-claude-code` | 外进程，**真 Claude Code**（官方 Agent SDK） | 否 |
| `subagent-dsh-sdk` | 外进程，另一个 dsh（JSON-RPC SDK） | 否 |

"子代理"的语义跨度从"本进程新起一个 agent"一直到"把任务委托给另一个商业产品的一轮对话"，全部藏在同一接口后。`inheritsParentContext` 被明确定义为**描述性而非可强制**的：只说孩子是否看到父对话历史，不说工具/服务/权限的继承。

## 两种生命周期：one-shot 与 continuable

**One-shot**：`start(name, request)` → 返回 caller 持有的 `SubagentRun` → `result` 解析出 `{ output, structured?, stopReason }` → 必须 `dispose()`。所有权转移边界清晰：fulfill 前 provider 负责失败时回滚并静默一切未发布资源；fulfill 后 caller 拥有 run。**孩子级失败以非 completed 的 stopReason 解析，只有 seam 无法表达的基础设施故障才允许 reject**。

**Continuable**：一个持久 Session + 至多一个进程内 **Activation**（"一段驻留纪元，不是请求/结果/取消/任务边界"）。核心设计判断：**Agent 收件箱是唯一的轮次队列**——continuation manager 只管驻留（residency），轮次排序与执行全归 agent loop。三种驻留状态从 Agent 静默性和持有孩子集合**推导**而来（running/waiting/settled），而非维护第二个状态机。冷恢复从持久化的 descriptor 重建，不再经过 provider——"持久 Session 已含初始前缀，折叠出的 descriptor 就是全部重建输入"。

能力协商在启动前完成：provider 通过 `capabilities` 宣告 `outputSchema`/`depthLimit`/`toolFilter`/`persona`，服务在**创建孩子之前**就拒绝不支持的请求。continuable 能力的检查更极致：`prepareContinuable?()` 方法的存在本身就是能力位。

## 委托策略：孩子的权限在委托边界钉死

`captureDelegatedPolicyOverrides(parent)` 在委托边界快照父会话的沙箱覆盖，并**把孩子的审批策略钉为 `'never'`**——无论父自己是什么策略。理由：被委托的孩子的每次审批请求"会等待一个没人看着的提示"，确定性拒绝优于挂起。策略以 `source: 'delegation'` 的事件写进**孩子自己的日志**（在 fork seed 之后，新策略赢过陈旧种子），孩子的有效策略从它自己的日志即可重建。父在创建后切换策略**永不追溯**改变已存在的孩子。

每个进程内孩子还收到一段固定的运行时上下文声明（`subagent:delegation`）：

> 你是被委托的子代理：权限范围在启动时已固定，不能从会话内部拓宽——需要审批的操作会被自动拒绝。任务需要超出范围的访问时，不要重试被拒的操作；在回复中说明限制，让委托方处理。

防御性设计的另一细节：`applyChildComposition(childCtx, parent, composition)` 把"父组合加入"做成**唯一入口的参数**——"把'不做这个 join 就组合孩子'在调用点上变得不可表达，正是这一个函数存在要防止的缺陷"。

## 结算投递（settlement delivery）：谁告诉父亲孩子干完了

这段是全 README 里最深思熟虑的部分。Activation 结算时，manager 向孩子的持久直接父亲投递通知，**无条件**——不管孩子有没有主动 report，因为"最需要交代的结局（token 上限、模型失败、取消、teardown）恰恰是孩子来不及选择的结局"。

两条顺序规则让投递"可靠而非侥幸"：

1. 发送发生在**释放孩子所有权之前**——父亲此刻还数着这个孩子，结构上不可能被判定为已结算
2. 父亲自己是 Activation 时，通知走与 report 相同的唤醒准入记账——同步发送与微任务准入之间的窗口不会被误判为静默

投递车道与 05 篇的后台任务同款三分：空闲父亲开新 turn；忙碌父亲被 steer 进最近的 step 边界（几个孩子同时结算花一步而非一轮）；**血统正在排水（draining）的父亲只 inject 不唤醒**——"teardown 期间唤醒会在 host 即将销毁的 agent 上花一个模型请求，且每层树各一次，因为每层的通知又会唤醒上一层"。

来源标记防冒名：通知携带 `{ kind: 'subagent-settled', form: 'notice' }`，与孩子亲笔的 `subagent-report` 是不同的 source kind——"transcript 永不把运行时写的话记在孩子名下"。

权限不对称也是有意的：`followup()` 要求确切的活体直接父亲；`interrupt()` 却接受持久父地址（人类）或任何活体祖先——"停止一个 turn 是幂等的且不投递内容，所以父 Agent 离线时人类仍能停住活着的孩子"。

## Claude Code / Codex 后端：把竞品产品包成 provider

2026-08-04 决策档案记录了两个第一方产品路由的完整契约：

- **只走官方集成面**：Codex 用 `codex app-server --stdio`，Claude Code 用官方 `@anthropic-ai/claude-agent-sdk`（钉 0.3.220）并把 host PATH 上解析到的 `claude` 可执行文件精确传入。被拒绝的替代方案点名："直接模型 HTTP、`codex exec`、手写 Claude CLI 协议——绕过官方可扩展集成面，无法证明原生配置/工具/审批/结果语义/teardown"
- **每次调用 = 全新产品进程 + 不可恢复的产品会话**；只接受自包含的纯文本任务；后台执行与产品选择**不是模型参数**（两个固定工具，各绑一个产品）
- **无人值守交互 fail closed**：Codex 的命令/文件审批选非批准决定（偏好 cancel）、不授予权限、拒绝 MCP elicitation；Claude Code 禁用 AskUserQuestion、不提供 canUseTool 回调。"没有合法无人值守响应的请求让 run 失败，而不是等一个 provider 不拥有的用户界面"
- **深度限制 `maxDepth: 'provider-managed'`**——递归策略留给产品自己，"不发送一个 provider 无法强制执行的限制"
- **证据分三层**：keyless 真产品测试（loopback 固定答案模型、字节精确断言、全进程树退出证明）+ Loader 组合测试（配置能加载且不启动产品）+ 带凭据 e2e（真产品从真 DeepSeek 服务拿唯一 nonce）。"产品替身不能替代任何一个产品层，手工挂载不能替代 Loader 层"
- 好玩的细节：Codex 0.147.0 说 Responses 协议而 DeepSeek 公开端点说 Chat Completions，credentialed e2e 用一个**仅回环的测试私有桥**转换——档案里诚实注明"这个桥不是生产代理，也不是 Codex 原生连接 DeepSeek 的证据"

**Placement 演化**（三篇档案接力）：先是 opt-in 组合 → 2026-08-10 移入共享 host → **2026-08-12 反转：生产 `dsh-base` 排除两个产品 provider**——"哪怕休眠的 provider 不起产品进程，它们的包仍进入每次生产 npm install"（Claude Agent SDK 不小）。要用就在 Profile 里显式安装挂载。决策档案的 supersede 链在这里是活的：新档案明确写"本决策取代 2026-08-10 的 placement"。

## 诚实的已知局限清单

README 结尾 10 条 Known Limitations，几条有代表性：ACP 孩子仍是 one-shot 且不可 trace 枚举（远程会话 id 需进 descriptor + 逐孩子协商 loadSession）；无跨进程驻留（Activation 收件箱不协调两个 harness 进程，需要持久邮箱 + 租约协议）；已接受但未落日志的消息崩溃即失、不自动重放；report 无持久邮箱（要求活体直接父亲，提供接受身份而非 exactly-once）。每条都写明了补齐它需要什么新契约。
