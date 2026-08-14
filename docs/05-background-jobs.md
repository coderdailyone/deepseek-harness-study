# 05 · 后台任务系统：run_in_background 与统一 jobs 运行时

> 研究对象：deepseek-harness @ `47f9438`。依据：`docs/tool-catalog.md`（生成）、`packages/jobs/jobs/README.md`、`.agents/notes/implemented/feature/2026-08-11-background-job-completion-wakes-an-idle-owner.md`、`.agents/notes/implemented/architecture/2026-06-20-generic-long-running-tool-runtime.md`。

## 模型面：不是独立工具，是 bash 的一个参数

dsh 的后台 shell 是 `bash` 工具的 `run_in_background: true` 参数。工具描述原文：

> Set `run_in_background: true` for long-running commands: the call returns a job id immediately; read its output with `job_output` and stop it with `job_kill`.

细节：

- `enableRunInBackground` 配置（默认 true）关闭时，**参数从 schema 里整个消失**——不是运行时报错，是模型根本看不到这个选项。schema 即能力面
- 每次 bash 调用是全新 shell（无状态持久），后台与否不改变这点；持久状态走另一个 seam（`tool-bash-persistent`/终端）

## 统一 jobs 运行时：三类生产者一套控制面

`ctx.jobs`（`dsh-jobs` 定义 / `dsh-jobs-local` 进程内实现）服务三类生产者，共用 `dsh-tool-jobs` 的三个模型面工具：

| 生产者 | 入口 |
|---|---|
| 后台 bash 命令 | `bash(run_in_background: true)` |
| 持久终端发送 | `terminal_send(run_in_background: true)` |
| 后台子代理 | `tool-subagent` 一次性后台委托 |

控制工具：

- **`job_output`** — 读输出。流式任务是单消费游标；终态任务幂等读
- **`job_list`** — 只列调用者拥有的 + 无主的任务（owner 隔离按 SessionId 比对；"`bash-1` 这类 id 可预测，所以这道栅栏就是边界"）
- **`job_kill`** — 先调生产者取消，成功才转 `stopping` 并标记终态投递已报告；取消抛出则任务保持运行

服务契约里几个讲究的点（`packages/jobs/jobs/README.md`）：

- `start()` 预检失败不留任何注册痕迹；成功返回后不再有可失败步骤
- 结算是 first-wins：一条终态记录、一轮被容错的监听者通知、释放全部 waiter
- 三种注册（controller/listener）都是 **owner 相对的**：一个组合没加载 controller，就不能借别的组合的控制面起后台工作
- `outputLimitBytes` 是生产者拥有的模型展示策略，注册表不改写生产者输出也不发明默认值
- 已知局限诚实列出：流输出单游标、前台不能升格为后台、契约是进程内的（跨进程后端要重塑身份/重启/所有权语义才能实现这个 seam）

## 完成通知的投递设计（2026-08-11 决策档案）

这篇 Agent Note 处理所有 harness 都要面对的问题：**模型说完话收工了，后台任务才跑完，通知给谁？**

原始缺口："任务完成会在会话内通知你，不要轮询"这句 prompt 只在模型还在工作时成立——完成通知走 `agent.inject()`（进下一步收件箱但不唤醒驱动器），turn 关了之后通知就停在没人认领的收件箱里。模型被告知不要轮询，然后什么也不会到来。

### 决策：按 owner 状态选投递道

- **忙碌的 owner → inject**（不变）：turn 循环在下一步收件箱非空时不能关闭，所以通知赶在检查前就延长当前 turn；几个任务同时结算花一步而非一轮
- **空闲的 owner → `followup()` 唤醒**：开新 turn 处理通知
- **被取消的 turn → 仍然 inject**：理由——"被取消的 turn 是用户按了停止，替用户重开一轮等于把中断洗成一个他们没要求的模型请求"

### 唤醒有预算，且预算不是时间

`maxConsecutiveWakes`（默认 3）限制一个 owner 连续被这样唤醒的轮数，超出后通知降级为 inject 等下一轮。预算存在的原因：**这条链是自激的**——被唤醒的 turn 可以启动新后台任务，其完成又唤醒它，没人看着（子代理结算不同：受模型生成的子代理数量天然限制）。

预算由**认领用户亲笔消息**恢复——认领而非到达，"因为那才是人的输入真正进入步骤的时刻"。本插件排队的通知永不恢复预算。`dsh run`（headless）不需要单独策略：唯一的用户消息在第一轮被认领后不再重复，预算单调花完，进程必然终止。

`completionDelivery: quiet` 恢复旧行为，为确定性 transcript 存在。

### 两个工程细节

- **teardown 认领报告**：`cancelForTeardown` 标记 `reported`，与 `kill()` 一致——否则一个唤醒式 reporter 会在 host 正在销毁的 agent 上每层 teardown 触发一次模型请求
- **完成最后公布**：`settle()` 原本在跑完成监听器*之后*才发布可见集变化，唤醒的 turn 的 `turn/start` 会赶在它所响应的结算提交之前落日志。改为 reporter 是结算的最后一个观察者

### Alternatives considered 点名了同行

这节是全仓库"记录打败了谁"纪律的精彩样本，直接对比了三家竞品的内部设计：

- **Codex 的 `trigger_turn`**（生产者声明的唤醒位，Kimi 的 admission 枚举同型）——"长期看是更好的形状（`tail -f` 流和两小时构建想要不同答案），但当前没有生产者需要区分，仓库要求公共接口有现任 owner"
- **Claude Code 的统一未经请求输入队列**（后台任务/cron/MCP push/hooks 合并进一个 drain 的优先级车道）——"dsh 的 inbox 本来就是那个队列（`agent/inbox/spliced` 的持久 splice），再加一层只为决定一个 bit"
- **Codex 的 `MailboxDeliveryPhase` 闩锁**（模型给出可见回答后拒绝重开 turn）——"这个闩锁正是本决策刻意反转的默认：模型说完话之后还能醒来正是全部意义所在，边界改用唤醒预算"
- **计数器之上再加墙钟窗口**——"对交互 agent 来说慢的场景恰恰是想要的场景：一小时构建结束、agent 恢复工作，这是 feature"

### 诚实登记的未闭合项

档案结尾记录了一个微任务竞态窗口（结算落在 turn 循环最后一次收件箱检查之后、驱动器提交空闲相位之前 → 读到 running 而 inject、无人唤醒），并说明关闭它需要 agent-loop 在最后一次认领前发布退休边界——"这是 core-agent 的决策，不是投递策略的决策"。缺陷登记连归属层级都标清楚了。
