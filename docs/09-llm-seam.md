# 09 · LLM 接缝：双适配器验证、失败事实与策略分离

> 研究对象：deepseek-harness @ `47f9438`。依据：`.agents/notes/implemented/architecture/2026-06-13-twin-llm-adapters.md`、`2026-06-21-bounded-llm-request-recovery.md`（本篇主要来源，含逐行链接的同行实现引证）、`2026-06-21-mandatory-app-attribution-headers.md`、`docs/subsystems/llm-streaming.md`。

## 词表：内容块与流协议

对话 = `Message[]`，消息 = 类型化内容块数组。块联合从 merge-extensible 的 `ContentBlockMap` 派生（text / reasoning / image / tool-call / tool-result），配一条准入门槛："新模态只有在适配器、UI、compaction、持久回放路径都支持它时才进这张表"——防止只有一半系统认识的块类型。

流协议 `StreamChunk`（block-start / text-delta / reasoning-delta / tool-call-delta / block-end / usage / finish）的关键约定：usage 在 finish 前发、finish 后无任何东西、工具调用 arguments 端到端保持原始 JSON 字符串、两条被认可的错误路径（`stream()` 抛出 *或* 以 `finish {kind:'error'|'aborted'}` 结束）消费者都必须处理。

## 双适配器：用两个真实现验证"中立"

最聪明的早期决策之一。**问题**：只对着一个适配器定义的"provider 中立"词表，会把那个实现的怪癖悄悄烤进契约——"那一个实现恰好做的任何事都成为事实规范，抽象在第二个 provider 到来前是未验证的，届时泄漏已昂贵到难修"。

**决策**：从第一天起对同一契约发布**两个刻意采用不同内部结构**的适配器：

- `dsh-llm-deepseek` — 直接 fetch + 仓库内翻译（SSE 解析交给 eventsource-parser）
- `dsh-llm-pi-ai` — 同一端点经 `@earendil-works/pi-ai` 库（它自己的事件词表）

强制执行的规则：**任何 StreamChunk 词表无法为两个实现同时表达的东西，都是核心词表的 bug**——立即被抓住而非等下一个 provider。上面那条"两条错误路径"约定正是库适配器暴露、单 fetch 适配器会隐藏的分歧。

被拒绝的替代方案：单适配器（"provider 中立"主张未经验证）；mock 第二适配器（"不打真 provider 的线上怪癖，证明不了什么。twin 是 real-on-real"）。代价诚实入账：适配器与带 key e2e 维护翻倍，换来持续的接缝中立性验证。

## 失败恢复：事实与策略的分离

`2026-06-21-bounded-llm-request-recovery.md` 是一篇教科书级的"关注点分离"设计：

**事实层**：`LlmFailure`（message / code / status? / providerRetryAfterMs? / requestId?）是唯一可序列化失败载荷。刻意**没有** `retryable`、`failover` 字段——"适配器报告事实，部署策略决定行动。同一个 429 在交互式组合里该重试，在成本封顶的批处理里该拒绝"。

**策略层**：`dsh-llm-retry` 是一个监听 `agent/request-error` 的函数插件——不加服务、不加循环分支。默认策略（两次瞬态重试、500ms 起步、10s 封顶、10% 抖动）的每个数字都有**逐行链接的同行引证**：OpenCode 的 2 次/500ms/10s、Pi 的 agent 级与 provider 级重试分离、Codex 的有界预算与 10% 抖动。工程决策精确到常数来源。

**一层拥有可见尝试**：适配器每次 `stream()` 恰好一次 provider 请求——pi-ai 适配器**删除**了库的重试配置并禁用库内重试。"SDK 预算不能乘上 agent 预算，每次瞬态重试都必须表现为一个关闭的失败步骤加一条 `llm/retry` 事件"。隐藏重试是被明确猎杀的对象。

**尝试在日志里天然分离**：失败的步骤可以留下 `assistant/chunk` 残渣，但永不追加 `assistant/message`、永不派发工具；重试关闭失败 turn、开新编号 turn、从持久 surface 重建请求。不需要第二套"响应生命周期"状态机——事件溯源结构免费提供了尝试隔离。重试前先落一条非 surface 的 `llm/retry` 事件（含轮次、失败步、策略键、延迟、失败事实）——"长时间的静默等待看起来像卡死的循环"，可观测性是一等需求。

**卡死流的界定**：每适配器一个 `streamIdleTimeoutMs`（默认五分钟，同 Codex）——可重臂的空闲看门狗，只在 `next()` 未决期间计时，"消费者思考时间不算 provider 空闲时间"；SSE 注释算传输活动但永不成为事件。边界测试要求证明超时**真的停掉底层请求**——"只 reject 消费者 promise 而让请求继续跑的计时器不满足契约"。

**Out of scope 也是设计**：自动 provider/model failover、语义输出修复、熔断器、跨 agent 重试预算——每条都写明了为什么现在不做（"没有当前消费者"、"无法证明语义兼容性"）。

## 归因头：标准优先的克制

`mandatory-app-attribution-headers` 档案做了一次完整的标准调查（RFC 9110 的 User-Agent/Referer/From 各节、OpenRouter 的 App Attribution、各家 coding agent 的实践），结论：**只强制标准的 `User-Agent`**，每个适配器必须有 mock server 断言头真的到线上的测试。OpenRouter 的 `HTTP-Referer`/`X-Title` 一族被明确拒绝——"那是 OpenRouter 特定的产品面头，不是 provider 中立的模型请求归因；哪怕请求指向 OpenRouter 也只发共享的 User-Agent"。防的是把某一家的约定当行业标准、然后把专有头泄漏给所有 provider 和无限期记日志的代理。

## 可借鉴的三个模式

1. **Twin 验证**：任何"中立契约"在只有一个实现时都是未验证的主张。付得起的话，第二个真实现是最便宜的契约测试
2. **事实/策略分离 + 单层拥有尝试**：失败载荷只有事实没有裁决；重试预算集中在一层，其他层（尤其 SDK）强制单次尝试——这是多层重试相乘灾难的唯一解
3. **常数带引证**：500ms/10s/10%/5min 每个数字标注出处。数字有据可查，后来者要改就得跟出处辩论，而不是跟拍脑袋辩论
