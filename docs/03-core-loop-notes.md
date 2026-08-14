# 03 · 核心代码阅读笔记：agent loop 三文件

> 研究对象：deepseek-harness @ `47f9438`。精读 `packages/core/agent-loop/src/`（index.ts 713 行、agent.ts 496 行、tool-calls.ts 289 行）与 `packages/core/agent/src/inbox.ts`（220 行）。整个 core/agent + core/agent-loop 合计约 3,300 行——"少而精"的实现哲学。

## index.ts（713 行）= 工厂与生命周期

最大篇幅不是业务逻辑，而是 **teardown 正确性**：

- 每个 agent 的销毁是一个 memoized 的逆序清算：停状态机 → 等静默（`whenIdle`）→ 退注册表 → 解卷 scope。且在发布*之前*就注册好——中途 unload 也能完整回滚（`prepare()` 的注释原文："registered BEFORE publication, so a mid-setup unload rolls everything back"）
- 取消信号三方熔合，各带各的 reason：调用者 cancel、所属 fiber unload、工厂 teardown（`AbortController` 组合 + `AbortSignal.any`）
- resume 路径的 load 与所有者生命周期赛跑："一个永不 settle 的持久化后端不能钉死会话身份"
- 配置驱动的 agent 启动失败有专门的瞬态事件 `agent-loop/config-start-failed`，让为该身份缓冲工作的消费者拒绝工作而非永久等待

这个文件就是 `docs/defensive-patterns.md`（生命周期/并发/子进程/teardown bug 类别）的活教材。

## agent.ts（496 行）= ReAct 循环本体

状态机只有三态：`idle | maintenance | running`。

**三种输入语义只是同一 inbox 的不同参数组合**：

| 方法 | 目标队列 | 唤醒？ | 用途 |
|---|---|---|---|
| `followup(msg)` | next-turn | 是 | 常规追问，开新 turn |
| `steer(msg)` | next-step | 是 | 转向：插进当前 turn 的下一步 |
| `inject(msg)` | next-step | 否 | 注入上下文：静静等别的消息触发时捎带 |

**每个请求从会话日志现场重建**（`session.deriveMessages()`），不存对话数组。请求配置（provider/model/system/tools）拼成 canonical header，变化时追加 `request/header` 事件——模型看到的一切配置变更都有审计痕迹。

细节见功力：

- max-tokens 结局是**粘性的**：一旦某步撞了输出上限，后续正常完成的步不能把 turn 结局降级
- 被清空的唤醒消息仍打开一个空 turn 边界——日志记下"尝试过"（"a rejected or empty first claim still closes a durable turn that spent no step, so the log records the attempt"）
- 唤醒闩锁（wake latch）语义：maintenance 或已中止的驱动器不能投递唤醒，闩住待收敛时重放；disposed 永不闩——teardown 不等任何模型轮次
- 错误结构化：`LlmError` 保留事实，其他一律经 `errorChain` 展平进 `UNKNOWN` 码的 turn 结局

## tool-calls.ts（289 行）= 并行工具调度器

- 并行调用进**有界滚动池**（`maxParallelToolCalls`，运行中可改，从设置读穿——"a committed change caps the next group without disturbing the one in flight"）；exclusive 调用形成屏障
- **dispatch 可乱序重叠，但 policy、结果提交、结果上下文严格按模型给出的顺序**（`commitReady` 只沿连续模型序推进）
- 每组开始前重读工具的并发模式——注册表热变更对未启动的调用生效，甚至能把 parallel 流中途变成屏障
- **中止时给未跑的调用补合成错误结果**（`appendSkippedToolCall`：`tool call aborted before dispatch`）——保证日志回放永远合法，每个模型调用都有配对的结果
- 调度器自身失败与工具失败分开：停止新派发、排空在途、以第一个失败拒绝，**不伪造恢复结果**

## inbox.ts（220 行）= 日志投影的收件箱

连收件箱都是事件溯源：

- 每次变更先写 `agent/inbox/spliced` 持久事件、再动内存投影——**未处理的用户输入跨重启可恢复**
- 构造时从日志重放全部 splice 事件重建状态；坏事件带 seq 报错
- splice 坐标按 JS `Array.prototype.splice` 语义归一化后才落日志，同步的 `session/event` 观察者看到 splice 前的列表、可用归一化坐标重建被删消息
- 待处理消息身份查重跨两个队列（next-turn + next-step），重复身份直接抛错

## 一个方法论侧写

代码里随处可见 `/* v8 ignore next -- 为什么这行不可达 */`——per-file 100% 覆盖率门禁逼着每一行不可测代码书面自辩。注释只写"合同"（行为/失败/时序/所有权），不写叙事（那由 `dsh-prose-standard` skill 管）。这些机械约束正是敢让 agent 大规模写码的原因：质量不靠人盯，靠门禁。
