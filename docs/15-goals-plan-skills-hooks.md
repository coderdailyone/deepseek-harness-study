# 15 · 驾驶层：Goal、Plan Mode、Skills、Hooks

> 研究对象：deepseek-harness @ `47f9438`。依据：`.agents/notes/implemented/feature/2026-07-19-model-facing-goal-tools.md`、`docs/tool-catalog.md` 的 plan-mode 条目与 headless profile 的 plan-mode 配置原文、`packages/skill/skill/README.md`、`packages/hooks/README.md`。

这层是"人怎么驾驭 agent"的产品面。四个子系统各有一个值得单独记住的设计。

## Goal 工具：机械权威与语义判断的分界

`create_goal` / `get_goal` / `update_goal` 三个工具管理同会话长期目标。精华在**执行时权威认证**——决策档案开篇就点破了为什么 prompt 不够："子代理、注入的插件消息、陈旧的模型轮次、恢复的会话都能产出相同的工具参数。文本能引导模型判断，但不能认证活的调用者、轮次与来源事件。"

权威规则全部是执行时检查，prompt 注入与手造参数都绕不过：

- 每次调用要求 `exec.agent` 是注册表里**确切的运行中对象**、当前驱动器发起者、且有开放的 turn
- create/edit/pause/resume 额外要求**当前 turn 里有直接人类消息**——用户来源是 **host 侧的证言**（每个 `followup()`/`steer()` 输入都必须带显式 source，host 给人类内容打 `{ kind: 'user' }`，非人类生产者自报身份）
- root 归属从**活 agent 图**推导而非持久 fork 祖先：恢复的 fork 可以获得直接人类权威，活着的孩子永远是子代理、不能动这些状态
- 自主目标轮只拿两个终态报告权（complete/blocked），且必须精确匹配当前目标的 id+revision+round——"它不获得编辑、暂停、恢复或替换人类目标的许可"

而机械与语义的分界写得非常清楚："运行时证明当前 turn 包含直接人类消息，**不证明人类的措辞在语义上是否值得创建或恢复——那个解释权留给模型**。"blocked 门槛同理：机械检查只数轮数（默认 3 轮起）与非空解释，"那些轮次是否遭遇同一阻塞条件的语义等价性仍是模型判断"。机器管认证，模型管理解，互不越界。

## Plan Mode：KV 缓存稳定性作为设计约束

`exit_plan_mode` 在规划**不活跃时也留在模型面 schema 里**——"转换不在计划策略变化之上再叠加工具目录搅动"。工具目录跨模式保持不变，plan-mode 规则以 prompt 声明"这些规则覆盖任何后来的工具描述；那些工具留在列表里只为保持请求形状稳定"。这是把 KV 缓存前缀稳定性当成一等设计约束的罕见样本（同族细节：skills 目录、审批策略上下文的渲染都标注 KV cache effect——README 模板里"KV Cache effect"是必填节）。

plan prompt 里还有一句对齐人机权界的话（我们在 dump-config 里看到原文）："用户的对话性同意——包括对你提问的确认回答——**不批准任何东西**、不结束 plan mode；把确认的决定折进计划，经 `exit_plan_mode` 提交。"批准只能走结构化通道，聊天不算数。

## Skills：分层注册表 + 无差别失效通知

`ctx.skills` 是纯 provider 注册表（不知道技能来自文件、内嵌数据还是 HTTP），沿用工具注册表确立的 **host+per-scope 分层**：全局层与 agent preset 层合并读取，重名时最近的层整体获胜。两个细节：

- `snapshot()` 带 `complete: false` 诚实标记——任何 provider 拒绝或自报不完整发现时，观察结果如实降级而非装作完整
- `skills/change` 是**不带目录也不带 diff 的失效通知**——"每个消费者用自己的查找选项重新 `snapshot()`"。不试图在事件里同步状态，只通知"该重新看了"，把一致性难题消解在源头

## Hooks：竞品协议桥接到自家扩展点

`packages/hooks/` 的定位一句话讲完："规范扩展面是 harness 自己的类型化拦截点——**native hook 就是那些扩展点上的一个普通 Cordis 插件**；这些包是把外部 shell-hook 协议翻译到同一表面上的**桥**。"`hooks-claude-code` 与 `hooks-codex` 各自指向已有的 `hooks.json`/settings，让用户的既有 hook 原样运行；共享一个 wire-protocol 库。

这个结构值得注意的是方向性：他们没有发明自己的 hook 配置格式再要求用户迁移，而是**兼容两家竞品的存量格式**、把执行落到自家管线的 `tools/pre-execute`/`tools/post-execute` waterfall 上（08 篇的守卫与审批与 hook 在同一管线排队）。存量生态的摩擦被桥消化，架构纯度由"桥也只是普通监听者"保住。

## 观感

四个子系统合起来是一套完整的"权界工程"：goal 用 host 证言划人类权威、plan 用结构化提交划批准通道、skills 用分层与诚实标记划可见性、hooks 用桥划生态兼容与架构纯度的边界。它们都遵循同一句潜台词——**凡是权力，必须机械可证；凡是理解，留给模型**。
