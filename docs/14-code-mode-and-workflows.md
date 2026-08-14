# 14 · Code Mode 与工作流：模型写程序，而不是发调用

> 研究对象：deepseek-harness @ `47f9438`。依据：`.agents/notes/implemented/feature/2026-06-15-code-mode.md`（本篇主要来源）、`2026-07-19-fresh-agent-ralph-workflow-tool.md`、`docs/tool-catalog.md` 的 `run_code`/`workflow`/`ralph` 条目。

## Code Mode：把工具目录变成一份 SDK

**动机**（引 Cloudflare 的 Code Mode 博文）：LLM 写代码比发工具调用强——它见过百万行真代码、只见过少量人造的工具调用轨迹。原生模式下多步工具工作既费 token 又串行：模型无法对结果集循环、按中间值分支、扇出、后处理，除非每次调用都付一个完整的模型往返，且**每个中间结果无论需不需要都拖回上下文**。

**决策**：`mode: 'native' | 'code' | 'both'` 是工具注册表的一等展示模式（cordis.yml 一行切换）。`code` 模式下注册表在线上只贡献一个保留传输工具 `run_code`，其余可见能力以**生成的 TypeScript 声明**（`.d.ts` + JSDoc，从 JSON Schema 编译）出现在 system prompt 里；模型写一段异步函数体，`await tools.name(args)` 调用，"只有你打印或返回的东西才回来——自己筛选"。

三个设计判断值得展开：

**1. 子调用重新进入完整守卫管线。** 程序里的每次 `tools.x()` 都以确定性 call id、外层调用为 parent，走完整的 `pre-execute → 守卫 → execute → post-execute` 管线——权限插件照样能拦，每次子派发落 `tool/code-dispatch-start`/`tool/code-dispatch` 日志事件对（日志可见、模型历史不可见）。Code Mode 改变的是**展示**，不是**授权**。子调用的 `additionalContexts` 经 `deferContext()` 延迟到外层结果之后注入——不破坏父调用/结果的相邻性。并发同样复用原生契约：提交序启动、并发安全的调用在 `maxParallelSubCalls`（默认 10）内重叠、exclusive 调用排干池子单独跑。

**2. 执行基底是能力接缝。** `ctx.codeRuntime` 对工具一无所知：拿到程序 + 命名异步绑定，跑，报 `{ value, logs, error? }`。语言与基底是后端属性——Python 后端就是另一个 provider 包 + 对应的 SDK 渲染器（后续档案已加 Python）。失败类型是**正交上报**的教科书应用：`exception | timeout | abort | worker-exit | invalid-output | output-limit` 六种互不冒充。

**3. worker-thread 实现假设对端怀有敌意。** 每次运行一个全新 Worker（无池化无跨运行状态——"程序的世界随 worker 死去，让状态渗漏不可表达"）；`env: {}` 真空环境（比子进程的擦洗规则更强）；类型剥离在宿主侧用 Node 内置 `stripTypeScriptTypes`（**保持位置**，运行时错误行号与模型源码对齐；语法失败绝不孵化 worker）；绑定桥的消息端口协议"假设一个敌对的对端，因为对端跑的是模型代码"——worker 侧命名空间对象 null-prototype 构建，叫 `__proto__`/`constructor` 的绑定只是普通自有属性；未知名、重复 id、结算后消息一律拒收。预算三条各自独立：`computeMs` 只计忙时（慢工具的 await 不背锅）、`maxWallMs` 封总时长、`maxOutputBytes` 只封外层输出（中间绑定值不设字节帽）。

**信任姿态的诚实辩护**：不加"我知道不安全"确认旗——因为仓库本来就船运 `dsh-bash-local`，它以**更多**环境权限执行任意模型 shell 命令。安全声明与既有基线对齐，不表演。

## Ralph：固定的"新鲜 agent 迭代"工作流

Ralph 模式 = 反复把同一目标交给**完全新鲜**的 worker，共享工作区当长期记忆，只带一小份显式交接，直到完成或到限。他们的实现选择很克制：

- **不进 loop、不进 goal driver、不进公共工作流语言**——"把一种策略耦合进无关的执行机器"是被点名拒绝的。`dsh-tool-ralph` 是独立 Consumer 包，一个固定的、可评审的脚本
- **子代理必须无继承**：provider 默认 `spawn`，调用前检查它存在、支持结构化输出、`inheritsParentContext: false`——fork 类 provider 直接 fail loud。"让孩子继承父对话会破坏上下文重置的意义，并让回放依赖不断增长的隐式前缀"
- **交接是带 schema 的结构化报告**：`{ status: continue|complete|blocked, summary, evidence, nextSteps, blocker }`，语义校验严格（continue 必须有 nextSteps 且无 blocker；complete 必须有 evidence；blocked 必须有具体 blocker），大小超限**失败而非静默截断**
- **预算对齐**：Ralph 的 `maxRounds`（默认 256）直接作为该次 workflow run 的子代理总数上限传给引擎——"固定循环的轮预算与通用的失控孩子后挡板不可能不一致"
- **结果措辞的分寸**：成功/受阻信封说"一个 worker 报告了此结果"，**不把它呈现为独立认证**——模型自报告不等于验证，措辞层面都不越界

层次：Ralph Run → Round → 新鲜子 Turn → Step。每轮恰好一个孩子，spawn 出独立会话、无种子、继承 cwd——"共享工作树是持久权威，父对话与先前孩子历史都不进请求"。

## 通用 workflow 工具

`workflow` 工具让模型写扇出编排脚本（worker 线程引擎执行，`agent()` 调用经 `ctx.subagents` 扇出）；Ralph 是它之上的**固定策略**特化。两者的分界照应了一个通用原则：**模型可编程的通用机制**（workflow 脚本、run_code 程序）与**部署拥有的固定策略**（Ralph 的 script/schema/caps，模型只能给 objective 和轮数）分开供货，各自演化。

## 观感

Code Mode 是全仓库"展示与授权分离"的极致样本：换一种模型面语言（JSON schema → TypeScript SDK），底下的守卫管线、日志事件、并发契约、取消语义一寸不让。而 Ralph 展示了如何把一个社区流行的 agent 模式（fresh-agent loop）落成**有边界的产品特性**——预算对齐、结构化交接、无继承强制、措辞不越权，每一条都是对该模式野生用法的一次收紧。
