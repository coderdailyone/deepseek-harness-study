# 11 · Host/Client 架构：类型驱动的 RPC 网关与浏览器端插件树

> 研究对象：deepseek-harness @ `47f9438`。依据：`docs/api-gateway.md`（164 行现状参考）、`docs/development.md` 的 TypeScript 工程布局节、`docs/cookbook/adding-a-conversation-node.md`。

## 为什么是两个编译面

仓库用两个隔离的 TypeScript 聚合（Host 与 Client），根因很技术但很硬：**两侧都对 Cordis 的 `Context` 接口做 declaration merging，同名 key 挂不同服务**——一个 `ts.Program` 同时看到两组 merge 就是类型冲突。冲突只存在于 Program 内（模块解析永不触发），所以解决方案是"两个聚合程序 + 一个跨两侧的 paths 门面"，并配三条纪律（base tsconfig 永不加 include、跨仓库脚本显式选一个面、每包只注册进一个聚合）。唯一例外 `api/remotes` 拥有 Host/Client 两张脸，workspace 约束门禁自动识别split 包并检查引用者选对了叶子。

**浏览器端也是 Cordis**：Web 客户端由插件组成，每个带 `dsh.client` 声明的包出一个 `lib/client.js`，`ctx.clientModules` 增量扫描组装 `__DSH_BOOT__` 入口图（我们跑通 Web UI 时在首页 HTML 里直接看到了这个 boot manifest——每个插件带自己的 client.js 与注入依赖）。"一切皆插件"贯穿到了浏览器。

## Typert：从 Host 类型直接生成 RPC 契约

业务服务用装饰器选择暴露给 Client 的方法：

- `@Remote('create')` — 调根 Host Context 上的服务；**复杂 Host 对象不过线**：签名里的 `Agent` 参数经 `TypertLookupMap` 变成 `agentId` 线上字段，网关在调用业务方法前把 id 解析回 Host 对象
- `@RemoteScope(key)` — 先把身份解析成 scoped Context，再从那个 Context 取服务调方法

构建期，Typert generator 以 Host `ts.Program` 为唯一种子做**严格分析**（方法必须是公开非静态实例方法、不能泛型、参数必须是必填的简单标识符——析构/默认值/rest/可选一律拒绝），产出四类工件：Host 运行时反射与严格调用描述符、Host 声明、可挂载的 Client 贡献（含运行时 codec）、Client 声明合并 + **declaration map**——编辑器里从 `ctx.remote.goals.create` 一跳直达 Host 源码上的 `@Remote` 方法，而不是停在生成的 `.d.ts`。

运行时，网关每次调用都从活注册表解析描述符与服务（不缓存业务对象），对 `args` 做逐字段精确匹配 + codec 校验 + 返回值校验——"缺 provider、未知身份、绑定不匹配、多参少参、schema 失败、方法缺失，都在进入业务代码之前或离开之后失败"。`agent`/`session` 的查找策略统一在一处：复用活 Agent、自动冷恢复普通会话、并发恢复去重、拒绝子代理路由拥有的身份。

**SRC 开发回退**的分寸感值得注意：Host 从源码经 tsx 启动时没有编译器插件，装饰器初始化器在 WeakMap 里记方法名，网关据此构造**较弱的临时描述符**（只解析参数名、只做 JSON 安全检查）——但 **Client 拒绝挂载 SRC 描述符**，它的类型与 codec 永远来自最近一次生成的工件。开发便利不动摇边界校验的强度，且"Host 热卸载一个严格端点也不降级为 SRC 推断，防止热卸载静默削弱校验"。

## 边界声明

`Remote 只处理一元调用`（一请求一结果）。会话事件流、分页、增量 reduce、实体子流"必须不伪装成 Remote 方法"——即使复用同一 Connection 也要走独立的数据协议。分层为 `remotes → gateway → connection → webserver`，Connection 拥有传输/关联/信封/取消，网关只拥有数据协议与业务派发——"未来换掉 Connection 载体不需要动 Remote 描述符"。

## Chat 视图：Conversation Node 的可回放纪律

cookbook 教程把 UI 行也纳入了事件溯源纪律。给 Chat 加一行业务节点的第一步不是写组件，是**设计可回放的事件家族**：

- 选一个稳定业务 id，每个相关事件都携带它——"client 永不把更新归给'最近一个未完成的'上下文"（禁止按时序猜归属）
- 每个 delta 必须在按 seq 升序回放时产生确定性 State，"不得依赖只活在内存里的东西"
- 优先整值检查点而非增量——窗口分页加载时，start 事件可能在窗外；必须在 start 未加载时渲染的产品，其终态事件要携带足够的整体回退状态，"不许靠扫描无关事件恢复"

UI 状态 = 日志的确定性折叠，和后端同一套物理定律。

## 观感

这一层是整个仓库里"工程密度"最高的地方：为了"业务服务写一个带装饰器的方法，浏览器端就获得带类型、带校验、可跳转源码的调用面"这一个开发体验，他们建了类型图生成器、双面构建管线、严格校验网关、SRC 降级路径和一整套边界声明。是否值得抄取决于项目规模——但"**校验强度不随开发模式降级**"和"**流不伪装成一元调用**"这两条边界纪律是任何规模都适用的。
