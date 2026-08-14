# 22 · Examples 体系与 Git 工作流：可跑的组合、官方的栈

> 研究对象：deepseek-harness @ `47f9438`。依据：`examples/AGENTS.md`、`.agents/notes/implemented/process/2026-08-02-native-github-stacks-and-optional-rebases.md`、根 `AGENTS.md` 的 PR 纪律条目。系列终篇。

## Examples：组合是产品的一部分

`examples/` 的定位精确到反直觉：**一个 workspace 成员但不是构建目标**——它是可运行与测试用 Cordis 配置的模块解析根，`examples/package.json` 声明那些配置加载的全部包（04 篇 pnpm-workspace 注释里"仅供依赖解析"的谜底）。

纪律三条：

- **可复用逻辑必须抽进 `packages/`**（那里有覆盖率与 README 门禁管着）；examples 只留 `cordis.yml` 接线、演示物料、e2e/snapshot 场景
- **每个 example 双冒烟强制**：keyless（真 Loader 起真 cordis.yml，抓 postmortem 0001 那类"手工挂载测试抓不到的非法导出"）+ with-key（真模型提示 + **验证外部状态而非模型的自述**）
- **cordis.yml 里只注释非显然的东西**：接线的坑、加载顺序后果、回放、安全边界、配置作用域——"不叙述看得见的条目"。连示例配置的注释密度都有编辑标准

"组合"在这个仓库里是被测试的一等产物：插件对不对是单元测试的事，**插在一起对不对**是 examples 的事。多数插件系统只测前者，坏在后者。

## Git 工作流：把栈交给平台

**依赖 PR 链必须用 GitHub 原生 stack 对象**（`gh stack link` 成栈、`gh stack merge` 整栈或前缀落地），拒绝的替代方案讲得清楚：只用分支链表示 = "GitHub 没有栈对象可依，无法展示顺序、跨层执行 trunk 规则、原子合并区间"，手动逐 PR 合并 + 重定向基底的旧流程被整个淘汰。几条配套纪律：

- **历史重写永远带租约**：精确 lease 或 `gh stack` 的受保护推送路径，远端动了就中止；**裸 `--force` 禁止**
- **merge-forward 与 rebase 都是合法刷新史**——"当保住已完成的冲突解决及其恢复点比紧凑历史更重要时，merge 检查点是有效选择"（不搞 rebase 原教旨）
- **`gh stack sync` 是"先检查后发布"的显式例外**（它一步完成 fetch+级联 rebase+push），代价规定死：每个被改写的层立即验证、证据过了才许合并；重写推送后 review 线程/批准/检查全部重审——"旧 commit OID 与行内锚点可能已失效"
- **冲突栈不自动解散**："本地分支推断不许覆盖共享的 GitHub 元数据"，混合作者与元数据冲突保留人类决策边界

配套三件套都是 skill/cookbook：`dsh-merging-stacked-prs`（落栈校验清单）、栈上回应评审的 cookbook（"修复落在引入它的那层"）、`dsh-pre-push-checks`（租约与 post-sync 证据）。加上 01 篇的标签分类法（每 PR 一个 `kind/*`、全部涉及 `area/*`、原生 Issue Type）与评审用的 `change-scope` 报告工具，agent 的整个 Git 操作面都是被制度化了的。

## 系列收官语

22 篇走完，这个仓库最终给我的印象可以压成三句话：

1. **它把"agent 大规模写码"当成一个系统工程问题而非提示词问题**——答案是门禁、档案、skill 与责任模型的四层制度（01/10/21），代码只是制度的产出物。
2. **它把"事件溯源 + 微内核 + 接缝"三件老武器组合出了 harness 领域的标准解**——log 即状态、扩展点即类型化事件、能力即三角色接缝（02–09），几乎每个子系统都是这三句话的分形。
3. **它最稀缺的品质是诚实的工程化**——强制承认局限的门禁、"验证世界而非自我报告"的测试观、"绿 ≠ 确认可靠"的门禁边界声明、postmortem 到守卫的闭环（10/12/16/21）。诚实不靠自觉，靠机器。

对读者的最后建议：如果只读三份上游原文，读 `docs/architecture.md`（129 行的地图）、`2026-07-06-sandbox.md`（39KB 的决策档案范本）、`docs/testing.md`（49 行的测试哲学）——一份看结构，一份看深度，一份看品格。
