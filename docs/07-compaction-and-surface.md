# 07 · Compaction 与 Surface：不可变日志上的历史手术

> 研究对象：deepseek-harness @ `47f9438`。依据：`.agents/notes/implemented/architecture/2026-06-18-session-surface.md`、`packages/compaction/compaction-basic/README.md`（164 行）、`docs/subsystems/compaction.md`。

事件日志 append-only 不可变（02 篇），那上下文压缩怎么办？答案分两层：**surface 机制**解决"如何在不改日志的前提下改写历史"，**compaction 引擎**解决"何时压、压什么、怎么压"。

## 第一层：Surface——日志之上的有序投影

**问题**（决策档案原文）：日志是权威，但历史操纵没有持久的共享机制。没有它，compaction 这类插件只能靠顺序敏感的监听器改写派生请求，不记录每次替换用了哪些事件；且每种新的历史操纵都得改 `deriveMessages()`。

**决策**：加一层 **surface**——事件序号的有序投影（产生 LLM 消息的事件子集），由日志里的 `surfaceOp` 标记维护。每个 `SessionEvent` 增加两个可选结构字段：

- **`sourceEventSeqs?: number[]`** — 引用的更早事件（如构成 `assistant/message` 的全部 `assistant/chunk`，或被 compaction 标记遮蔽的 surface 节点）。"没有这些引用序号，回放无法验证一次 replace-range 操作点名了它移除的每个事件"
- **`surfaceOp?: SurfaceOp`** — 两种操作：`'append'`（常规尾追加）或 `{ op: 'replace', start, end }`（含端点遮蔽区间、插入本事件）

关键性质：**被遮蔽的事件留在日志里，只是不再出现在 surface 上**。压缩不删除任何东西——完整历史永远可考，只有投影变了。持久化零改动（JSONL 直接序列化顶层字段），崩溃修复合成的 `tool/result` 关闭事件也带合法 surface 标记，保证再水化后 surface 有效。

不变量在始终生效的 seed/append 边界校验：引用必须唯一、更早、已知；replace 端点必须存在于当前 surface 序；`sourceEventSeqs` 必须覆盖每个被遮蔽节点。被拒绝的替代方案：逐插件包裹 `agent/request`（监听器顺序脆弱、无持久记录）、半开区间（端点是事件序号，单条替换 `start===end` 用闭区间读起来自然）、链表节点+序号映射（生产代码没人读前驱链接，单数组同渐进复杂度、只有一种表示要校验）。

## 第二层：BasicCompactionEngine——压缩策略

`ctx.compaction` seam 的基础实现，几个环节：

**测量**：单例 `ctx.tokenMeter` 在一个消费日志版本号上为最新 canonical 信封 + 当前 surface 定价——步边界压力测量包含真实 system prompt、工具、路由、助手补全、工具结果、缓冲上下文与转向消息，不是拍脑袋估算。精度诚实声明：没有可复用的 provider usage 时退化为字符数+结构开销启发式。

**两条入口**：主动压力（`agent/pre-step` 监听器在请求派生前查压力，默认阈值 = 路由模型上下文窗 × 0.8）与 **provider 确认的溢出恢复**（`agent/request-error` 进入；"provider 已经证实压缩是必要的"，所以绕过常规压力与保留策略，直接剪枝 + 一次最大平衡头部缩减）。

**先免模型剪枝**：可选的 `ctx.toolResultPruner` 先把超大工具结果做**可回放的单节点 surface 替换**，重新测量，压力安全就完全跳过摘要调用——省一次模型调用。

**保留策略**：压最老的整 surface 单元、保留最近尾部、工具调用/结果对切割平衡（不留孤儿调用）。"turn 边界不保护失控 turn 内的旧步骤"——防御跑飞的单 turn 吃满窗口。

**摘要调用的 KV 缓存设计**（最精彩的工程细节）：摘要请求**逐字节回放对话自己的 system prompt、工具 schema、被遮蔽区消息**，只在末尾追加一条压缩指令作为最终用户消息——"复用 provider 的暖前缀缓存而不是让它失效，只有指令和摘要输出是未缓存的"。适配器还会带上 `x-deepseek-harness-compact: 1` 归因头（不动模型可见 body）。只有返回的文本进入检查点——排除 reasoning（防私有推理泄漏）与工具调用（防孤儿调用）。

**检查点提示词**是结构化八节 Markdown（Primary Request and Intent / Key Technical Concepts / Files and Code / Errors and Fixes / Pending Jobs / Current Work / Next Step / Critical Context），规则含"精确保留文件路径、命令、错误串、标识符、数值、函数签名"、"不得提及本次摘要请求或上下文被压缩过"、"已有 `<compacted-summary>` 块是先前检查点，合并勿照抄"。与 Claude Code 的 compact 提示词是同一门手艺，可对照研究。

**收敛与失败**：拒绝不缩小其来源的摘要；重试耗尽仍超阈值则抛出。**日志里活的未配对 `compaction/start` 就是持久锁**（bracket-first 区域事务），失败的关闭刻意留下阻塞性孤儿——宁可卡住也不静默放行。摘要失败时"保留最新持久 surface"：自动路径告警后带着超预算的完整历史继续跑——**压缩失败不毁对话**。

**配置分层**：顶层策略是所有路由模型的默认，`modelPolicies` 按精确 provider/model 对做部分覆盖——同一插件同时服务大小窗模型。非法配置（重复目标、互斥保留形式、retainRatio ≥ thresholdRatio）在插件加载时 fail loud。

## 值得抄的三个点

1. **投影而非变异**：压缩 = 日志追加一条 replace 标记。完整历史可审计、检查点可回滚、`sourceEventSeqs` 让每次替换自证覆盖完整——这是"不可变日志 + 可变视图"的教科书实现
2. **免模型剪枝前置**：先做零成本的结构化剪枝再决定要不要花钱摘要，省调用又降低摘要输入量
3. **摘要请求的前缀缓存复用**：摘要模型看到的就是对话模型上一个请求的逐字节前缀 + 一条尾部指令——把辅助调用的成本压到最低的同时避免了"为摘要另构造一套输入"的失真
