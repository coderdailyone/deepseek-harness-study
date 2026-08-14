# 18 · 小接缝巡礼：spill、telemetry、E2B、attachment、session-query、terminal

> 研究对象：deepseek-harness @ `47f9438`。依据：各包 README 与 capability-seams 生成目录。这些外围接缝每个都小，但各有一个值得单独记住的设计点。

## spill：超大工具输出的溢写策略

`dsh-spill-policy` 是一个 `tools/post-execute` 转换器：纯文本结果超过 `maxInlineBytes` 时，全文存进 `ctx.spillStore`、模型面替换为"头尾预览 + 定位符 + 取回提示"。三个细节：

- **职责拆到不能再拆**：policy 只决定"何时溢写"并组装通知；预览是 `output-retention` 的事、存储是 `spillStore` 的事——一个 `tools/post-execute` 监听器就完成组合
- **预算含通知自身**：通知的字节成本从预算里预留，预览缩小让位——"模型面结果永不超帽"，没有"帽 + 一点点"的模糊地带
- **豁免清单讲得出道理**：跳过嵌套执行（其持久副本由 dispatch 日志侧封顶）、跳过 `read`（避免"读→溢写→再读"循环）、跳过非 accept 决定（block 的纠正反馈要原样过去）；**omitted 配置 = 整个策略不注册**——又一个"不宣传兑现不了的杠杆"

## session-telemetry：遥测的克制三连

- **`emit()` 必须无阻塞入队**（它同步跑在 `session/event` 里）；`session/flush` 的并行等待"永不等遥测"——可观测性不许拖慢主路径
- **redact waterfall 自己不带任何规则**："没挂监听器时记录原样到达后端，导出数据的干净程度恰好等于部署挂的规则"——脱敏责任显式归部署，框架不给虚假安全感；脱敏只作用于出站副本，"canonical 日志永不改写"
- **共享披露是接缝词表**：`full | feedback-only | disabled` 三态由后端强制声明，`/feedback` 确认界面如实展示——用户被告知的是握手事实（"handoff 不是 delivery"措辞都抠了）
- 递交游标用 `WeakMap<Session, seq>` 水位——文档明写这是"**对'注册皆效果'纪律的一次刻意的、狭窄的例外**"并给了三条理由（条目随会话死、值是单调水位、丢了也不算错）；at-most-once 的代价也直说："resume 不补投上个进程没送到的记录——要补投需求就要 outbox，不是回放"

## E2B：seam 主张的活证明

`packages/e2b/` 是 02 篇那句"一次 provider 替换改变整个产品"的实证 POC：`ctx.e2b` 管沙箱生命周期，`fs-e2b` 与 `subprocess-e2b` 分别实现文件系统与子进程两个基础接缝——**`bash-local`、`terminal-bash`、`lsp-stdio` 零 fork**，因为它们本来就把一切执行世界操作委托给 `ctx.fs`/`ctx.subprocess`，挂上两个 E2B 适配器，可变工作就整体搬进了同一个远程沙箱。边界声明同样清楚：harness 进程、Cordis 对象、模型调用、会话状态、持久化都不动——移动的只是"执行世界"。

## attachment：图像的持久引用模型

`ctx.attachments`（持久二进制附件存储）的一句话契约信息量很大："**host 在会话事件之前提交已接受的图像；provider 适配器把已授权的持久引用解析为 provider 原生内容**"。即：日志里存的是持久引用而非字节，模型请求组装时才由适配器换成各家格式——回放不依赖易失内存，编码细节留在适配器层。`read_image` 工具的注册与执行分层同样讲究：没有 `ctx.attachments` 就不注册；schema 与路由无关，但**执行时拒绝**当前路由模型未声明图像输入的调用。

## session-query：读路径的三层分工

接口层（`session-query`）提供精确读取、过滤、trace；SQLite 后端**增加**全文检索对账、排名、snippet、游标世代；而模型面消费者（`tool-session-query` 的五个只读工具）拥有工作区权限判定并**隐藏 provider 游标**、每个结果都从不可变的调用方会话授权。搜索能力增强不改变权限模型——谁能看什么由消费面钉死，后端只管找得快。

## terminal：注册表与机制分离

`ctx.terminals` 注册表拥有"**精确 Agent 的会话身份与清理**"，后端（`terminal-bash`，PTY 经 node-pty）拥有终端机制，`tool-terminal` 暴露六个 owner 隔离的模型工具。schema 的负空间同样声明了：TUI、命名键序列、BEL、resize、自动启动、跨 agent 共享"都不在 schema 里"——持久终端只承诺它能管好的那部分。

## 巡礼总结

六个小接缝重复着全仓库的三句话：**职责拆到单一**（spill 三分、terminal 注册表/机制/工具三分）、**负空间声明**（telemetry 不带脱敏规则、terminal 不做 TUI、E2B 不动 harness 进程）、**能力门控展示**（attachment 缺席则 read_image 不注册、spill 配置缺席则策略不注册）。小部件不小气，处处是同一套纪律的分形。
