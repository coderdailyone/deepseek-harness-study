# 20 · Vendoring 与发布工程：完全拥有框架层

> 研究对象：deepseek-harness @ `47f9438`。依据：`vendor/README.md`（18 条局部修改的完整台账）、`.agents/notes/implemented/process/2026-08-06-in-repository-landlock-release.md`、`2026-08-13-public-vendor-and-native-sequences.md`。

## 为什么源码级 vendor 整个框架

Cordis 框架及其地基库（cosmokit/schemastery/loader/include/hmr 等九个包）不走 npm 依赖，而是**源码复制进 monorepo**——"让 harness 完全拥有它的框架层（可审计、可打补丁、可钉死）"。全部重命名进 `@deepseek-ai` scope（发布 harness 就发布这层框架，用上游名字发布会抢注 registry）；目录名与上游版本号刻意不动，manifest 表仍读作上游快照。`verify-vendored-links` 门禁断言每个 vendored 名字都解析到 workspace link 且没有 registry 副本混入。

## 局部修改台账：18 条，每条有理由有测试

`vendor/README.md` 的军规："**这份日志必须穷尽——与上游的每一处分歧都必须列出**。"18 条修改从琐碎到深刻：

- 琐碎端：类型放宽一行（`writeTask?: NodeJS.Timeout | undefined`，exactOptionalPropertyTypes 需要）也入账并标注"type-only; no behavior change"
- 深刻端：**#6 `cordis/src/fiber.ts` 生命周期硬化**——本地修复三个重入销毁缺口（setup 内发起的 unload 等待 setup 与全部清理、UNLOADING 期拒绝新效果注册防止清理期注册逃出快照、teardown 通知逐观察者容错）。03 篇读到的 agent-loop teardown 纪律，根子在这里被他们改进了框架本身
- **#12 一个 exit 13 无诊断的死锁**：Include 子树变更不可重入 + HMR 初扫重入刷新 → 序列化队列 + `ignoreInitial: true`，台账里连死锁链路都写全了
- **#11 `--dump-config` 的诚实性**：把补丁算法抽成导出的纯函数，"配置工具永不允许重新实现（并漂移于）补丁算法"——我们在 04 篇用过的 dump-config 精确性原来是这条修改买来的；顺手还修了上游 bug（后 patch 无法配置先 patch 插入的行）
- 每条几乎都以 "Covered by `<具体测试文件>`" 收尾——**vendored 代码的修改与自有代码同等被测试钉住**

同步纪律：上游 sync 后重放或退役每条已记录修改、`pnpm run rescope-vendor --apply` 重施改名、重跑 test+build。JSDoc 增补（#7）标注了退役条件："上游化之后撤销此条"。

## Landlock 的仓内发布：一个 release 工程样本

08 篇那个 ~300 行 C 启动器的发布路径本身就是一篇制度设计（2026-08-06 档案）：

- **消灭镜像仓库**：源码、消费者、CI、发布同一仓库同一 lockfile——"一个 PR 可以同时改启动器契约与其消费者并一起测试；一个 release tag 同时标识源码、消费者集成、构建指令与被测 tarball"
- **平台选择保留**：公共分发是 1 个 JS 入口包 + linux-x64/arm64 两个二进制包（npm 的 `os`/`cpu` 字段选择性安装），"仓库所有权与 npm 包布局是两个独立选择"——单包塞全部二进制被明确拒绝
- **发布次序防悬空**：平台 tarball 先于依赖它们的入口 tarball 发布；专属 `landlock-run-vX.Y.Z` tag 防 monorepo 内 release 家族冲突
- **rehearsal 不许 registry 代打**：打包安装排演用**当前 checkout 的本地 tarball** 装进外部纯 Node 消费者，证明装出来的启动器可执行、**与原生构建逐字节一致**、ELF 架构正确——然后才测 confinement
- 诚实收尾：npm 无跨包事务，失败发布可能留下部分版本——操作规程写明"检查 registry、只发缺失的 tarball，不许原样重跑工作流"

## 分级公开：access 是序列属性，不是 scope 属性

2026-08-13 档案把发布访问级建模得很清楚：三个发布序列（vendored 框架九包 / native 三包 / dsh 家族 221 包）各自持有 `publishConfig.access`，workspace 约束门禁把每个 manifest 钉在其序列的级别上——"scope 不会漂移：新 vendor 包留在 restricted、或 dsh 成员翻成 public，都过不了约束检查"。推导链条也严密：dsh 家族每个包 peer 依赖 vendored 框架、sandbox 依赖 Landlock 入口——"**受限的依赖才是真正挡住公共消费者的东西**"，所以这两个序列必须先公开。**没有任何发布路径传 `--access` 旗**——"一个旗服务不了意见相左的序列，且旗会覆盖拥有该事实的 manifest"。

我们研究的 head（#2519 `feat/npm-public`）正是最后一步：dsh 家族 221 包转公开发布（`npx @deepseek-ai/dsh` 从此可用）——三个序列分级推进到全公开的收官。

## 观感

这一篇的三个系统共享同一个思想：**分歧必须有台账**（vendor 修改日志穷尽 + 测试钉住）、**事实住在拥有它的 manifest 里**（access 不进旗标、版本匹配进约束门禁）、**发布的每一步在发布前排演**（本地 tarball 装进真消费者、逐字节比对）。对任何需要 fork/vendor 依赖或维护多包发布的项目，"18 条台账"的格式——修改内容 + 理由 + 测试覆盖 + 退役条件——是可以直接抄走的模板。
